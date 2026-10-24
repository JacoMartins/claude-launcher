package io.github.jacomartins.canyonui.ui.drawer

import android.graphics.Rect
import androidx.annotation.DrawableRes
import androidx.compose.foundation.Image
import androidx.compose.foundation.background
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.lazy.grid.GridItemSpan
import androidx.compose.foundation.lazy.grid.LazyGridScope
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.layout.LayoutCoordinates
import androidx.compose.ui.layout.boundsInWindow
import androidx.compose.ui.layout.onPlaced
import androidx.compose.ui.res.painterResource
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.search.ContactApp
import io.github.jacomartins.canyonui.search.ContactResult
import io.github.jacomartins.canyonui.search.HandOff
import io.github.jacomartins.canyonui.search.QuickAction
import io.github.jacomartins.canyonui.search.ShortcutResult
import io.github.jacomartins.canyonui.search.SearchExtras
import io.github.jacomartins.canyonui.ui.design.PrimaryButton
import io.github.jacomartins.canyonui.ui.design.SecondaryButton
import io.github.jacomartins.canyonui.ui.design.Spacing
import kotlin.math.roundToInt

/*
 * Canyon search's results besides apps, as full-width rows in the Apps screen's grid: above the
 * apps the calculator, below them shortcuts, contacts, settings and hand-offs to other apps. An
 * empty search-only screen lists recent searches under the suggested apps.
 */

private fun LazyGridScope.full(key: String, content: @Composable () -> Unit) =
    item(key = key, span = { GridItemSpan(maxLineSpan) }, contentType = key.substringBefore(':')) { content() }

/**
 * Above the apps: the calculator's answer, when the search is a calculation, or a line saying
 * [nothing] matched on the phone (other apps can still be searched, below).
 */
fun LazyGridScope.searchTop(extras: SearchExtras, model: DrawerModel, actions: DrawerActions, nothing: Boolean) {
    if (nothing) {
        full("none:") {
            Text(
                stringResource(R.string.no_search_results, model.query.trim()),
                style = MaterialTheme.typography.bodyLarge,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
                modifier = Modifier.padding(horizontal = Spacing.sm, vertical = Spacing.md),
            )
        }
    }
    // What the query asks for: a command, a person to message, an address. The first one is
    // what Go does, so it stands out.
    extras.top.forEachIndexed { i, a ->
        full("top:$i:${a.title}") {
            ResultRow(title = a.title, subtitle = a.subtitle, image = a.image, icon = a.icon, highlight = i == 0 && extras.goFirst) { b ->
                model.rememberQuery()
                model.search.record(a.key, a.person)
                runAction(a, b, actions)
            }
        }
    }
    val calc = extras.calc ?: return
    full("calc:") { CalcRow(extras.query, calc) { actions.copyText(calc) } }
}

/** Does what a [QuickAction] says. */
fun runAction(a: QuickAction, bounds: Rect?, actions: DrawerActions) {
    when (val run = a.run) {
        is QuickAction.Run.Open -> actions.openIntent(run.intent, bounds)
        is QuickAction.Run.Call -> actions.placeCall(run.number)
        is QuickAction.Run.Shortcut -> actions.openShortcut(run.shortcut, bounds)
        QuickAction.Run.AllowContacts -> actions.requestContacts()
    }
}

/** Below the apps: the other kinds of result for the query. */
fun LazyGridScope.searchBottom(extras: SearchExtras, askContacts: Boolean, model: DrawerModel, actions: DrawerActions) {
    if (extras.query.isEmpty()) return
    fun opened() = model.rememberQuery()
    for (h in extras.direct) {
        full("direct:${h.kind}") { HandOffRow(h) { b -> opened(); actions.openIntent(h.intent(), b) } }
    }
    if (extras.shortcuts.isNotEmpty()) {
        full("title:shortcuts") { SectionTitle(stringResource(R.string.search_shortcuts)) }
        for (s in extras.shortcuts) {
            full("shortcut:${s.info.`package`}/${s.info.id}") {
                ResultRow(title = s.label, subtitle = s.appLabel, image = s.icon, icon = R.drawable.ph_lightning) { b ->
                    opened(); model.search.record(ShortcutResult.key(s), null)
                    actions.openShortcut(s, b)
                }
            }
        }
    }
    if (extras.contacts.isNotEmpty() || askContacts) {
        full("title:contacts") { SectionTitle(stringResource(R.string.search_contacts)) }
        for (c in extras.contacts) {
            full("contact:${c.id}") {
                ResultRow(
                    title = c.name,
                    image = c.photo,
                    letter = c.name.firstOrNull { it.isLetterOrDigit() }?.uppercaseChar(),
                    trailing = { ContactButtons(c, actions) { key -> opened(); model.search.record(key, c.lookupKey) } },
                ) { b ->
                    opened(); model.search.record(null, c.lookupKey)
                    actions.openIntent(android.content.Intent(android.content.Intent.ACTION_VIEW, c.uri), b)
                }
            }
        }
        if (askContacts) full("ask:contacts") { AskContacts(onAllow = actions::requestContacts, onDecline = model.search::contactsDeclined) }
    }
    if (extras.settings.isNotEmpty()) {
        full("title:settings") { SectionTitle(stringResource(R.string.search_settings)) }
        for (p in extras.settings) {
            full("setting:${p.label}") {
                val context = androidx.compose.ui.platform.LocalContext.current
                ResultRow(title = stringResource(p.label), icon = p.icon) { b ->
                    opened(); model.search.record(p.key, null)
                    actions.openIntent(p.intent(context), b)
                }
            }
        }
    }
    full("title:other") { SectionTitle(stringResource(R.string.search_other_apps)) }
    for (h in extras.handOffs) {
        full("handoff:${h.kind}") { HandOffRow(h) { b -> opened(); actions.openIntent(h.intent(), b) } }
    }
}

/** An empty search-only screen: the title over the suggested apps. */
fun LazyGridScope.searchStart(suggested: Boolean) {
    if (suggested) full("title:suggested") { SectionTitle(stringResource(R.string.search_suggested)) }
}

/** An empty search-only screen: recent searches, under the suggested apps. */
fun LazyGridScope.searchRecents(recents: List<String>, model: DrawerModel) {
    if (recents.isEmpty()) return
    full("title:recent") {
        SectionTitle(stringResource(R.string.search_recent), action = stringResource(R.string.search_clear), onAction = model.search::clearRecents)
    }
    for (q in recents) {
        full("recent:$q") { ResultRow(title = q, icon = R.drawable.ph_clock_counter_clockwise) { model.query = q } }
    }
}

@Composable
private fun SectionTitle(text: String, action: String? = null, onAction: (() -> Unit)? = null) {
    Row(
        Modifier.fillMaxWidth().padding(start = Spacing.sm, end = Spacing.xs, top = Spacing.md, bottom = Spacing.xs),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Text(
            text,
            style = MaterialTheme.typography.titleSmall.copy(fontSize = 14.sp, fontWeight = FontWeight.Bold),
            color = MaterialTheme.colorScheme.onSurfaceVariant,
            modifier = Modifier.weight(1f),
        )
        if (action != null && onAction != null) {
            Text(
                action,
                style = MaterialTheme.typography.labelLarge,
                color = MaterialTheme.colorScheme.primary,
                modifier = Modifier.clip(CircleShape).clickable(role = Role.Button, onClick = onAction).padding(horizontal = Spacing.smMd, vertical = Spacing.xs),
            )
        }
    }
}

/** Holds a row's layout coordinates, for the opening animation's start bounds. */
private class RowCoords {
    var value: LayoutCoordinates? = null

    fun bounds(): Rect? {
        val c = value?.takeIf { it.isAttached } ?: return null
        val b = c.boundsInWindow()
        return Rect(b.left.roundToInt(), b.top.roundToInt(), b.right.roundToInt(), b.bottom.roundToInt())
    }
}

/**
 * One result: a picture ([image], else a [letter] or a Phosphor [icon] on a round plate), a title
 * and an optional subtitle.
 */
@Composable
private fun ResultRow(
    title: String,
    subtitle: String? = null,
    image: ImageBitmap? = null,
    @DrawableRes icon: Int? = null,
    letter: Char? = null,
    /** The row Go opens: a light tint of the accent behind it. */
    highlight: Boolean = false,
    trailing: (@Composable () -> Unit)? = null,
    onClick: (Rect?) -> Unit,
) {
    val coords = remember { RowCoords() }
    val colors = MaterialTheme.colorScheme
    Row(
        Modifier
            .fillMaxWidth()
            .onPlaced { coords.value = it }
            .clip(RoundedCornerShape(20.dp))
            .then(if (highlight) Modifier.background(colors.primary.copy(alpha = 0.14f)) else Modifier)
            .clickable(role = Role.Button) { onClick(coords.bounds()) }
            .padding(horizontal = Spacing.sm, vertical = Spacing.sm),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Box(
            Modifier.size(40.dp).clip(CircleShape).then(if (image == null) Modifier.background(colors.surfaceContainerHighest.copy(alpha = 0.7f)) else Modifier),
            contentAlignment = Alignment.Center,
        ) {
            when {
                image != null -> Image(image, contentDescription = null, contentScale = ContentScale.Crop, modifier = Modifier.size(40.dp))
                letter != null -> Text(letter.toString(), style = MaterialTheme.typography.titleMedium, color = colors.onSurface)
                icon != null -> Icon(painterResource(icon), contentDescription = null, tint = colors.onSurface, modifier = Modifier.size(20.dp))
            }
        }
        Spacer(Modifier.width(Spacing.smMd))
        Column(Modifier.weight(1f)) {
            Text(title, style = MaterialTheme.typography.bodyLarge, color = colors.onSurface, maxLines = 1, overflow = TextOverflow.Ellipsis)
            if (subtitle != null) {
                Text(subtitle, style = MaterialTheme.typography.bodySmall, color = colors.onSurfaceVariant, maxLines = 1, overflow = TextOverflow.Ellipsis)
            }
        }
        trailing?.invoke()
    }
}

/** A contact's quick buttons: message (WhatsApp or another chat app, else SMS), call, video call. */
@Composable
private fun ContactButtons(c: ContactResult, actions: DrawerActions, opened: (key: String) -> Unit) {
    val chat = c.app(ContactApp.Kind.CHAT)
    val video = c.app(ContactApp.Kind.VIDEO)
    val phone = c.phone
    Row {
        if (chat != null || phone != null) {
            val label = stringResource(R.string.action_message, c.name) + (chat?.let { " · " + it.appLabel } ?: "")
            SmallIconButton(R.drawable.ph_chat_circle, label) {
                opened(c.chatKey(chat))
                actions.openIntent(chat?.intent() ?: android.content.Intent(android.content.Intent.ACTION_SENDTO, android.net.Uri.fromParts("smsto", phone, null)), null)
            }
        }
        if (phone != null) SmallIconButton(R.drawable.ph_phone, stringResource(R.string.action_call, c.name)) { opened(c.callKey); actions.placeCall(phone) }
        if (video != null) {
            SmallIconButton(R.drawable.ph_video_camera, stringResource(R.string.action_video, c.name) + " · " + video.appLabel) { opened(c.videoKey(video)); actions.openIntent(video.intent(), null) }
        }
    }
}

@Composable
private fun SmallIconButton(@DrawableRes icon: Int, label: String, onClick: () -> Unit) {
    Box(
        Modifier.size(40.dp).clip(CircleShape).clickable(role = Role.Button, onClickLabel = label, onClick = onClick),
        contentAlignment = Alignment.Center,
    ) {
        Icon(painterResource(icon), contentDescription = label, tint = MaterialTheme.colorScheme.onSurfaceVariant, modifier = Modifier.size(20.dp))
    }
}

@Composable
private fun HandOffRow(h: HandOff, onClick: (Rect?) -> Unit) {
    val (icon, title) = when (h.kind) {
        HandOff.Kind.CALL -> R.drawable.ph_phone to stringResource(R.string.handoff_call, h.value)
        HandOff.Kind.EMAIL -> R.drawable.ph_envelope_simple to stringResource(R.string.handoff_email, h.value)
        HandOff.Kind.LINK -> R.drawable.ph_link to stringResource(R.string.handoff_link, h.value.substringAfter("://"))
        HandOff.Kind.WEB -> R.drawable.ph_globe to stringResource(R.string.handoff_web, h.value)
        HandOff.Kind.STORE -> R.drawable.ph_storefront to stringResource(R.string.handoff_store)
        HandOff.Kind.MAPS -> R.drawable.ph_map_pin to stringResource(R.string.handoff_maps)
    }
    ResultRow(title = title, icon = icon, onClick = onClick)
}

@Composable
private fun CalcRow(expression: String, result: String, onCopy: () -> Unit) {
    val colors = MaterialTheme.colorScheme
    Row(
        Modifier
            .fillMaxWidth()
            .padding(bottom = Spacing.xs)
            .clip(RoundedCornerShape(20.dp))
            .clickable(role = Role.Button, onClickLabel = stringResource(R.string.search_copy_result), onClick = onCopy)
            .padding(horizontal = Spacing.sm, vertical = Spacing.sm),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Box(Modifier.size(40.dp).clip(CircleShape).background(colors.surfaceContainerHighest.copy(alpha = 0.7f)), contentAlignment = Alignment.Center) {
            Icon(painterResource(R.drawable.ph_calculator), contentDescription = null, tint = colors.onSurface, modifier = Modifier.size(20.dp))
        }
        Spacer(Modifier.width(Spacing.smMd))
        Column(Modifier.weight(1f)) {
            Text("= $result", style = MaterialTheme.typography.headlineSmall, color = colors.onSurface, maxLines = 1, overflow = TextOverflow.Ellipsis)
            Text(expression + " · " + stringResource(R.string.search_copy_result), style = MaterialTheme.typography.bodySmall, color = colors.onSurfaceVariant, maxLines = 1, overflow = TextOverflow.Ellipsis)
        }
        Icon(painterResource(R.drawable.ph_copy), contentDescription = null, tint = colors.onSurfaceVariant, modifier = Modifier.padding(end = Spacing.xs).size(20.dp))
    }
}

@Composable
private fun AskContacts(onAllow: () -> Unit, onDecline: () -> Unit) {
    val colors = MaterialTheme.colorScheme
    Column(Modifier.fillMaxWidth().padding(horizontal = Spacing.sm, vertical = Spacing.xs)) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Box(Modifier.size(40.dp).clip(CircleShape).background(colors.surfaceContainerHighest.copy(alpha = 0.7f)), contentAlignment = Alignment.Center) {
                Icon(painterResource(R.drawable.ph_user), contentDescription = null, tint = colors.onSurface, modifier = Modifier.size(20.dp))
            }
            Spacer(Modifier.width(Spacing.smMd))
            Column(Modifier.weight(1f)) {
                Text(stringResource(R.string.search_contacts_ask), style = MaterialTheme.typography.bodyLarge, color = colors.onSurface)
                Text(stringResource(R.string.search_contacts_ask_summary), style = MaterialTheme.typography.bodySmall, color = colors.onSurfaceVariant)
            }
        }
        Row(Modifier.fillMaxWidth().padding(top = Spacing.sm), horizontalArrangement = Arrangement.spacedBy(Spacing.sm, Alignment.End)) {
            SecondaryButton(stringResource(R.string.search_not_now), onDecline)
            PrimaryButton(stringResource(R.string.search_allow), onAllow)
        }
    }
}
