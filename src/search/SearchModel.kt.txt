package io.github.jacomartins.canyonui.search

import android.Manifest
import android.app.SearchManager
import android.content.Context
import android.content.Intent
import android.content.pm.LauncherApps
import android.content.pm.PackageManager
import android.content.pm.ShortcutInfo
import android.net.Uri
import android.os.Process
import android.provider.AlarmClock
import android.provider.MediaStore
import androidx.compose.ui.graphics.asImageBitmap
import androidx.core.content.edit
import androidx.core.graphics.drawable.toBitmap
import androidx.core.net.toUri
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.model.AppSearch
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.FlowPreview
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.debounce
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.stateIn

/**
 * Canyon search's sources besides apps, all on the phone: commands ("call ana", "stories on
 * instagram", "timer 5 min"), contacts (once allowed) with their chat and call apps, app
 * shortcuts, settings pages, addresses, the calculator and hand-offs to other apps. Also keeps
 * recent searches.
 */
@OptIn(FlowPreview::class)
class SearchModel(
    context: Context,
    scope: CoroutineScope,
    query: Flow<String>,
    /** Labels of the apps shown in the drawer, by package (hidden apps left out). */
    private val appLabels: () -> Map<String, String>,
) {
    private val app = context.applicationContext
    private val launcherApps = app.getSystemService(LauncherApps::class.java)
    private val prefs = app.getSharedPreferences("search", Context.MODE_PRIVATE)
    private val iconPx = (40 * app.resources.displayMetrics.density).toInt()
    private val people = ContactLookup(app, appLabels)
    private val stats = ActionStats(app)

    /** A result was opened: it (and its person) count for what comes first next time. */
    fun record(key: String?, person: String?) {
        if (key != null) stats.record(key)
        if (person != null) stats.record(ActionStats.person(person))
    }

    /** How much [a] has been used: the action itself, and a little its person. */
    private fun used(a: QuickAction): Double =
        (a.key?.let { stats.score(it) } ?: 0.0) + 0.25 * (a.person?.let { stats.score(ActionStats.person(it)) } ?: 0.0)

    /** The most used first; unused ones keep their order. */
    private fun List<QuickAction>.byUse(): List<QuickAction> = withIndex().sortedWith(compareByDescending<IndexedValue<QuickAction>> { used(it.value) }.thenBy { it.index }).map { it.value }

    private val _recents = MutableStateFlow(loadRecents())
    val recents: StateFlow<List<String>> = _recents.asStateFlow()

    /** Contacts may be read (the permission is granted). Re-checked when search opens. */
    private val contactsAllowed = MutableStateFlow(hasContacts())
    private val _askContacts = MutableStateFlow(false)
    /** Offer to allow contacts: not yet granted and not turned down with "Not now". */
    val askContacts: StateFlow<Boolean> = _askContacts.asStateFlow()

    @Volatile private var shortcuts: List<ShortcutInfo>? = null

    // Kept while search is open (cleared when it opens again): shortcut icons already loaded, and
    // whether an app answers an intent ("its own search"), so typing does not redo them.
    private val shortcutIcons = android.util.LruCache<String, androidx.compose.ui.graphics.ImageBitmap>(ICON_CACHE)
    private val answers = java.util.concurrent.ConcurrentHashMap<String, Boolean>()

    val extras: StateFlow<SearchExtras> = combine(
        query.map { it.trim() }.distinctUntilChanged().debounce(EXTRAS_DEBOUNCE_MS),
        contactsAllowed,
    ) { q, contacts ->
        when {
            q.isEmpty() -> SearchExtras()
            // Whatever another app's data holds, a bad answer never stops the search.
            else -> try {
                build(q, contacts)
            } catch (e: RuntimeException) {
                SearchExtras(query = q, handOffs = listOf(HandOff(HandOff.Kind.WEB, q)))
            }
        }
    }
        .flowOn(Dispatchers.IO)
        .stateIn(scope, SharingStarted.Eagerly, SearchExtras())

    /** Search is opening: shortcuts and the contacts permission may have changed since last time. */
    fun refresh() {
        shortcuts = null
        shortcutIcons.evictAll()
        answers.clear()
        contactsAllowed.value = hasContacts()
        _askContacts.value = !contactsAllowed.value && !prefs.getBoolean(KEY_NO_CONTACTS, false)
    }

    fun contactsDeclined() {
        prefs.edit { putBoolean(KEY_NO_CONTACTS, true) }
        _askContacts.value = false
    }

    /** A result was opened for [q]: it goes to the top of the recent searches. */
    fun remember(q: String) {
        val t = q.trim()
        if (t.isEmpty()) return
        val list = (listOf(t) + _recents.value.filterNot { it.equals(t, ignoreCase = true) }).take(MAX_RECENTS)
        _recents.value = list
        prefs.edit { putString(KEY_RECENTS, list.joinToString(SEP)) }
    }

    fun clearRecents() {
        _recents.value = emptyList()
        prefs.edit { remove(KEY_RECENTS) }
    }

    private fun loadRecents(): List<String> =
        prefs.getString(KEY_RECENTS, null)?.split(SEP)?.filter { it.isNotBlank() }.orEmpty()

    private fun hasContacts() = app.checkSelfPermission(Manifest.permission.READ_CONTACTS) == PackageManager.PERMISSION_GRANTED

    private fun build(q: String, contactsOn: Boolean): SearchExtras {
        val calc = Calc.evaluate(q)?.let { Calc.format(it) }
        val labels = appLabels()
        val command = Commands.parse(q)?.let { resolveInApp(it, labels) } ?: appFirst(q, labels)
        val top = ArrayList<QuickAction>()
        var goFirst = false
        if (command != null) {
            top += commandActions(command, contactsOn, labels).byUse()
            goFirst = top.isNotEmpty()
        }
        // A plain name: the contacts it matches, with their chat app.
        val contacts = if (contactsOn && calc == null && command == null) safeFind(q, MAX_CONTACTS) else emptyList()
        if (command == null && calc == null) {
            QueryKind.address(q)?.let { addr ->
                top += (destinationActions(addr, labels) + listOf(
                    QuickAction(str(R.string.action_open_maps, addr), null, R.drawable.ph_map_pin, run = open(geo(addr)), key = "maps:open"),
                    QuickAction(str(R.string.action_directions, addr), null, R.drawable.ph_navigation_arrow, run = open(directions(addr)), key = "maps:directions"),
                )).byUse()
                goFirst = true
            }
            val forPerson = recommended(q, contacts, labels)
            if (forPerson.isNotEmpty()) {
                top.addAll(0, forPerson)
                goFirst = true
            }
        }
        val direct = if (command != null) emptyList() else buildList {
            QueryKind.phone(q)?.let { add(HandOff(HandOff.Kind.CALL, it)) }
            QueryKind.email(q)?.let { add(HandOff(HandOff.Kind.EMAIL, it)) }
            QueryKind.url(q)?.let { add(HandOff(HandOff.Kind.LINK, it)) }
        }
        val handOffs = listOf(HandOff.Kind.WEB, HandOff.Kind.STORE, HandOff.Kind.MAPS).map { HandOff(it, q) }
        return SearchExtras(
            query = q,
            top = top,
            goFirst = goFirst,
            calc = calc,
            shortcuts = if (calc != null || command != null) emptyList() else findShortcuts(q, labels),
            // The person recommended at the top is not listed again below.
            contacts = contacts.filter { c -> top.none { it.person == c.lookupKey } },
            settings = if (calc != null || command != null) emptyList() else findSettings(q),
            direct = direct,
            handOffs = handOffs,
        )
    }

    private fun str(id: Int, vararg args: Any): String = app.getString(id, *args)

    private fun open(intent: Intent) = QuickAction.Run.Open(intent)

    private fun safeFind(name: String, limit: Int): List<ContactResult> = try {
        // The people contacted most first among those matching (namesakes, nicknames).
        people.find(name, limit).withIndex()
            .sortedWith(compareByDescending<IndexedValue<ContactResult>> { stats.score(ActionStats.person(it.value.lookupKey)) }.thenBy { it.index })
            .map { it.value }
    } catch (e: SecurityException) {
        contactsAllowed.value = false
        emptyList()
    }

    // --- Commands ----------------------------------------------------------------------------

    /** An InApp command whose app part names no app is no command (plain text with "on" in it). */
    private fun resolveInApp(c: Command, labels: Map<String, String>): Command? =
        if (c is Command.InApp && findApp(c.app, labels) == null) null else c

    /** "instagram stories": an app's name, then what to do in it. */
    private fun appFirst(q: String, labels: Map<String, String>): Command? {
        val nq = AppSearch.normalize(q)
        val hit = labels.values
            .map { it to AppSearch.normalize(it) }
            .filter { (_, n) -> n.isNotEmpty() && nq.startsWith("$n ") && nq.length > n.length + 1 }
            .maxByOrNull { it.second.length } ?: return null
        val rest = q.trim().substring(hit.first.length.coerceAtMost(q.trim().length)).trim()
        return if (rest.isEmpty()) null else Command.InApp(rest, hit.first)
    }

    /** The package of the app called [name] (exact, else the shortest label starting with it). */
    private fun findApp(name: String, labels: Map<String, String>): String? {
        val n = AppSearch.normalize(name)
        if (n.isEmpty()) return null
        labels.entries.firstOrNull { AppSearch.normalize(it.value) == n }?.let { return it.key }
        return labels.entries.filter { AppSearch.rank(it.value, n) == 0 }.minByOrNull { it.value.length }?.key
    }

    private fun commandActions(c: Command, contactsOn: Boolean, labels: Map<String, String>): List<QuickAction> = when (c) {
        is Command.Call -> people(c.who, contactsOn) { p ->
            when (c.via) {
                Command.Via.WHATSAPP -> p.app(ContactApp.Kind.CALL, null)?.takeIf { it.pkg in ContactResult.WHATSAPP }?.let {
                    QuickAction(str(R.string.action_call, p.name), it.appLabel, R.drawable.ph_phone, p.photo, open(it.intent()), key = "${p.callKey}:${it.pkg}", person = p.lookupKey)
                }
                else -> call(p)
            }
        } ?: QueryKind.phone(c.who)?.let { listOf(QuickAction(str(R.string.action_call, it), null, R.drawable.ph_phone, run = QuickAction.Run.Call(it))) }.orEmpty()
        is Command.VideoCall -> people(c.who, contactsOn) { p ->
            video(p)
        }.orEmpty()
        is Command.Message -> people(c.who, contactsOn) { p -> chat(p, c.via) }
            ?: listOfNotNull(numberChat(c.who, c.via))
        is Command.Email -> QueryKind.email(c.who)?.let { listOf(QuickAction(str(R.string.action_email, it), null, R.drawable.ph_envelope_simple, run = open(HandOff(HandOff.Kind.EMAIL, it).intent()))) }
            ?: people(c.who, contactsOn) { p ->
                p.emails.firstOrNull()?.let {
                    QuickAction(str(R.string.action_email, p.name), it, R.drawable.ph_envelope_simple, p.photo, open(HandOff(HandOff.Kind.EMAIL, it).intent()), key = "email:${p.lookupKey}", person = p.lookupKey)
                }
            }.orEmpty()
        is Command.Play -> {
            val pkg = c.app?.let { findApp(it, labels) }
            val what = if (c.app != null && pkg == null) "${c.what} ${c.app}" else c.what
            val intent = Intent(MediaStore.INTENT_ACTION_MEDIA_PLAY_FROM_SEARCH)
                .putExtra(MediaStore.EXTRA_MEDIA_FOCUS, "vnd.android.cursor.item/*")
                .putExtra(SearchManager.QUERY, what)
                .apply { if (pkg != null) setPackage(pkg) }
            listOf(QuickAction(str(R.string.action_play, what), pkg?.let { labels[it] }, R.drawable.ph_play, run = open(intent)))
        }
        is Command.Directions -> listOf(
            QuickAction(str(R.string.action_directions, c.place), null, R.drawable.ph_navigation_arrow, run = open(directions(c.place)), key = "maps:directions"),
            QuickAction(str(R.string.action_open_maps, c.place), null, R.drawable.ph_map_pin, run = open(geo(c.place)), key = "maps:open"),
        ) + destinationActions(c.place, labels)
        is Command.Timer -> listOf(
            QuickAction(
                str(R.string.action_timer, durationText(c.seconds)), null, R.drawable.ph_timer,
                run = open(Intent(AlarmClock.ACTION_SET_TIMER).putExtra(AlarmClock.EXTRA_LENGTH, c.seconds).putExtra(AlarmClock.EXTRA_SKIP_UI, false)),
            ),
        )
        is Command.Alarm -> listOf(
            QuickAction(
                str(R.string.action_alarm, String.format(java.util.Locale.ROOT, "%d:%02d", c.hour, c.minute)), null, R.drawable.ph_alarm,
                run = open(Intent(AlarmClock.ACTION_SET_ALARM).putExtra(AlarmClock.EXTRA_HOUR, c.hour).putExtra(AlarmClock.EXTRA_MINUTES, c.minute)),
            ),
        )
        is Command.InApp -> inApp(c, contactsOn, labels)
    }

    /**
     * The contacts named [who] turned into actions by [action] (up to three); null when [who] is
     * no contact (a number, or contacts are not allowed — then the action asks for them).
     */
    private fun people(who: String, contactsOn: Boolean, action: (ContactResult) -> QuickAction?): List<QuickAction>? {
        if (QueryKind.phone(who) != null) return null
        if (!contactsOn) return listOf(QuickAction(str(R.string.action_allow_contacts, who), null, R.drawable.ph_user, run = QuickAction.Run.AllowContacts))
        return safeFind(who, 3).mapNotNull(action).takeIf { it.isNotEmpty() }
    }

    /** Message a contact: WhatsApp (or another chat app) when it has one, else SMS. */
    private fun chat(p: ContactResult, via: Command.Via?): QuickAction? {
        val viaApp = when (via) {
            Command.Via.SMS -> null
            Command.Via.WHATSAPP -> p.app(ContactApp.Kind.CHAT)?.takeIf { it.pkg in ContactResult.WHATSAPP }
            null -> p.app(ContactApp.Kind.CHAT)
        }
        if (viaApp != null) {
            return QuickAction(str(R.string.action_message, p.name), viaApp.appLabel, R.drawable.ph_chat_circle, p.photo, open(viaApp.intent()), key = p.chatKey(viaApp), person = p.lookupKey)
        }
        if (via == Command.Via.WHATSAPP) return null
        val number = p.phone ?: return null
        return QuickAction(str(R.string.action_message, p.name), str(R.string.search_sms), R.drawable.ph_chat_circle, p.photo, open(sms(number)), key = p.chatKey(null), person = p.lookupKey)
    }

    private fun call(p: ContactResult): QuickAction? = p.phone?.let {
        QuickAction(str(R.string.action_call, p.name), it, R.drawable.ph_phone, p.photo, QuickAction.Run.Call(it), key = p.callKey, person = p.lookupKey)
    }

    private fun video(p: ContactResult): QuickAction? = p.app(ContactApp.Kind.VIDEO)?.let {
        QuickAction(str(R.string.action_video, p.name), it.appLabel, R.drawable.ph_video_camera, p.photo, open(it.intent()), key = p.videoKey(it), person = p.lookupKey)
    }

    /** Ride and navigation apps installed (Uber, Waze, Maps) taking [place] as the destination. */
    private fun destinationActions(place: String, labels: Map<String, String>): List<QuickAction> =
        AppLinks.destinations(labels.keys).map { link -> linkAction(link, place, labels[link.pkg] ?: "") }

    private fun linkAction(link: AppLinks.Link, text: String, appLabel: String): QuickAction = when (link.kind) {
        AppLinks.Kind.RIDE -> QuickAction(str(R.string.action_ride, text), appLabel, R.drawable.ph_car, run = open(link.intent(text)), key = "link:${link.pkg}")
        AppLinks.Kind.DRIVE -> QuickAction(str(R.string.action_drive, text), appLabel, R.drawable.ph_navigation_arrow, run = open(link.intent(text)), key = "link:${link.pkg}")
        AppLinks.Kind.SEARCH -> QuickAction(str(R.string.action_in_app, text, appLabel), null, R.drawable.ph_search, run = open(link.intent(text)), key = "link:${link.pkg}")
    }

    /** "zap 11 91234 5678": a chat with a number that is not a contact. */
    private fun numberChat(who: String, via: Command.Via?): QuickAction? {
        val number = QueryKind.phone(who) ?: return null
        val intent = if (via == Command.Via.WHATSAPP) {
            Intent(Intent.ACTION_VIEW, "https://wa.me/${number.filter { it.isDigit() }}".toUri())
        } else {
            sms(number)
        }
        return QuickAction(str(R.string.action_message, number), null, R.drawable.ph_chat_circle, run = open(intent))
    }

    private fun sms(number: String) = Intent(Intent.ACTION_SENDTO, Uri.fromParts("smsto", number, null))
    private fun geo(place: String) = Intent(Intent.ACTION_VIEW, ("geo:0,0?q=" + Uri.encode(place)).toUri())
    private fun directions(place: String) =
        Intent(Intent.ACTION_VIEW, ("https://www.google.com/maps/dir/?api=1&destination=" + Uri.encode(place)).toUri())

    private fun durationText(seconds: Int): String {
        val h = seconds / 3600
        val m = seconds % 3600 / 60
        val s = seconds % 60
        return listOfNotNull(
            h.takeIf { it > 0 }?.let { "$it h" },
            m.takeIf { it > 0 }?.let { "$it min" },
            s.takeIf { it > 0 }?.let { "$it s" },
        ).joinToString(" ")
    }

    /** "stories on instagram": that app's shortcuts for it, its chats with a person, its own search. */
    private fun inApp(c: Command.InApp, contactsOn: Boolean, labels: Map<String, String>): List<QuickAction> {
        val pkg = findApp(c.app, labels) ?: return emptyList()
        val appLabel = labels[pkg] ?: return emptyList()
        val out = ArrayList<QuickAction>()
        // The app's own deep links: "uber <address>" rides there, "waze <address>" drives there.
        AppLinks.forApp(pkg).forEach { out += linkAction(it, c.what, appLabel) }
        // A person in a chat app ("ana on whatsapp").
        if (contactsOn) {
            for (p in safeFind(c.what, 3)) {
                p.apps.firstOrNull { it.pkg == pkg && it.kind == ContactApp.Kind.CHAT }?.let {
                    out += QuickAction(str(R.string.action_message, p.name), appLabel, R.drawable.ph_chat_circle, p.photo, open(it.intent()), key = p.chatKey(it), person = p.lookupKey)
                }
            }
        }
        // The app's own shortcuts, with the words people use for them.
        val terms = AppSearch.normalize(c.what).let { w -> listOf(w) + w.split(' ').flatMap { Commands.SYNONYMS[it].orEmpty() } }.distinct()
        allShortcuts().filter { it.`package` == pkg }
            .mapNotNull { s ->
                val label = (s.shortLabel ?: s.longLabel)?.toString() ?: return@mapNotNull null
                val names = listOfNotNull(label, s.longLabel?.toString())
                val r = terms.withIndex().flatMap { (i, t) -> names.map { n -> AppSearch.rank(n, t).let { if (it < 0) -1 else it + i * 3 } } }
                    .filter { it >= 0 }.minOrNull() ?: return@mapNotNull null
                r to shortcutResult(s, label, appLabel)
            }
            .sortedBy { it.first }
            .take(3)
            .forEach { (_, s) -> out += QuickAction(s.label, appLabel, R.drawable.ph_lightning, s.icon, QuickAction.Run.Shortcut(s), key = ShortcutResult.key(s)) }
        // Its own search, and playing it when the app plays media.
        val search = Intent(Intent.ACTION_SEARCH).setPackage(pkg).putExtra(SearchManager.QUERY, c.what)
        // (Not twice when a deep link already searches the app.)
        if (out.none { it.key == "link:$pkg" } && handles(search)) {
            out += QuickAction(str(R.string.action_in_app, c.what, appLabel), null, R.drawable.ph_search, run = open(search), key = "search:$pkg")
        }
        val play = Intent(MediaStore.INTENT_ACTION_MEDIA_PLAY_FROM_SEARCH).setPackage(pkg)
            .putExtra(MediaStore.EXTRA_MEDIA_FOCUS, "vnd.android.cursor.item/*").putExtra(SearchManager.QUERY, c.what)
        if (handles(play)) out += QuickAction(str(R.string.action_play, c.what), appLabel, R.drawable.ph_play, run = open(play), key = "play:$pkg")
        return out
    }

    private fun handles(intent: Intent): Boolean = answers.getOrPut("${intent.action}|${intent.`package`}") {
        app.packageManager.queryIntentActivities(intent, 0).isNotEmpty()
    }

    /**
     * Typing a person's name: what is usually done with them comes first (message, call or video
     * call; a message when nothing was done yet), when the name is a clear match (it or one of its
     * words starts with the query) and no app's name starts with the query (unless the query is
     * the person's whole first name). The person contacted most wins among several matches.
     */
    private fun recommended(q: String, contacts: List<ContactResult>, labels: Map<String, String>): List<QuickAction> {
        val nq = AppSearch.normalize(q)
        if (nq.length < 2) return emptyList()
        val p = contacts.firstOrNull { AppSearch.rank(it.name, nq) in 0..1 } ?: return emptyList()
        val name = AppSearch.normalize(p.name)
        val exact = name == nq || name.substringBefore(' ') == nq
        if (!exact && labels.values.any { AppSearch.rank(it, nq) == 0 }) return emptyList()
        return listOfNotNull(chat(p, null), call(p), video(p)).byUse()
    }

    // --- Other sources -------------------------------------------------------------------------

    private fun findSettings(q: String): List<SettingPage> {
        val nq = AppSearch.normalize(q)
        return SettingsCatalog.pages
            .mapNotNull { p ->
                val label = app.getString(p.label)
                val byLabel = AppSearch.rank(label, nq)
                val byWord = p.keywords.split(' ').minOfOrNull { k -> AppSearch.rank(k, nq).let { if (it < 0) Int.MAX_VALUE else it + 3 } } ?: Int.MAX_VALUE
                val r = if (byLabel >= 0) byLabel else byWord
                if (r == Int.MAX_VALUE) null else r to p
            }
            .sortedWith(compareByDescending<Pair<Int, SettingPage>> { stats.score(it.second.key) }.thenBy { it.first })
            .take(MAX_SETTINGS)
            .map { it.second }
    }

    /** Shortcuts need the launcher to be the default home app; otherwise there are none. */
    private fun allShortcuts(): List<ShortcutInfo> {
        shortcuts?.let { return it }
        val list = try {
            if (!launcherApps.hasShortcutHostPermission()) emptyList()
            else {
                val query = LauncherApps.ShortcutQuery().setQueryFlags(
                    LauncherApps.ShortcutQuery.FLAG_MATCH_DYNAMIC or LauncherApps.ShortcutQuery.FLAG_MATCH_MANIFEST or
                        LauncherApps.ShortcutQuery.FLAG_MATCH_PINNED,
                )
                launcherApps.getShortcuts(query, Process.myUserHandle()).orEmpty().filter { it.isEnabled }
            }
        } catch (e: SecurityException) {
            emptyList()
        } catch (e: IllegalStateException) {
            emptyList()
        }
        shortcuts = list
        return list
    }

    private fun shortcutResult(s: ShortcutInfo, label: String, appLabel: String): ShortcutResult {
        val key = ShortcutResult.key(s)
        val icon = shortcutIcons.get(key) ?: try {
            launcherApps.getShortcutIconDrawable(s, app.resources.displayMetrics.densityDpi)?.toBitmap(iconPx, iconPx)?.asImageBitmap()
                ?.also { shortcutIcons.put(key, it) }
        } catch (e: Exception) {
            null
        }
        return ShortcutResult(s, label, appLabel, icon)
    }

    private fun findShortcuts(q: String, labels: Map<String, String>): List<ShortcutResult> {
        val nq = AppSearch.normalize(q)
        return allShortcuts()
            .mapNotNull { s ->
                val appLabel = labels[s.`package`] ?: return@mapNotNull null
                val label = (s.shortLabel ?: s.longLabel)?.toString() ?: return@mapNotNull null
                val r = AppSearch.rank(label, nq).takeIf { it >= 0 }
                    ?: s.longLabel?.toString()?.let { AppSearch.rank(it, nq) }?.takeIf { it >= 0 }
                    ?: return@mapNotNull null
                Triple(r, s, label to appLabel)
            }
            // The ones opened often first, then by how well they match.
            .sortedWith(compareByDescending<Triple<Int, ShortcutInfo, Pair<String, String>>> { stats.score(ShortcutResult.key(it.second)) }.thenBy { it.first })
            .distinctBy { it.second.`package` + "/" + it.second.id }
            .take(MAX_SHORTCUTS)
            .map { (_, s, l) -> shortcutResult(s, l.first, l.second) }
    }

    private companion object {
        const val EXTRAS_DEBOUNCE_MS = 60L
        const val MAX_RECENTS = 8
        const val MAX_SETTINGS = 4
        const val MAX_SHORTCUTS = 6
        const val MAX_CONTACTS = 5
        const val ICON_CACHE = 64
        const val KEY_RECENTS = "recents"
        const val KEY_NO_CONTACTS = "contacts_declined"
        const val SEP = "\n"
    }
}
