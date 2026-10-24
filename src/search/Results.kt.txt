package io.github.jacomartins.canyonui.search

import android.app.SearchManager
import android.content.ContentUris
import android.content.Intent
import android.content.pm.ShortcutInfo
import android.net.Uri
import android.provider.ContactsContract
import androidx.annotation.DrawableRes
import androidx.compose.ui.graphics.ImageBitmap
import androidx.core.net.toUri

/** An app shortcut ("New message", "Scan QR code") of an installed app. */
class ShortcutResult(val info: ShortcutInfo, val label: String, val appLabel: String, val icon: ImageBitmap?) {
    companion object {
        /** Its [ActionStats] key. */
        fun key(s: ShortcutResult) = key(s.info)
        fun key(info: ShortcutInfo) = "shortcut:${info.`package`}/${info.id}"
    }
}

/** A way to reach a contact that an app added to the contact ("Message +55 11..." from WhatsApp). */
class ContactApp(
    val dataId: Long,
    val mimeType: String,
    val pkg: String,
    val appLabel: String,
    val kind: Kind,
) {
    enum class Kind { CHAT, CALL, VIDEO }

    /** Opens that app on this contact, as the phone's contacts app does. */
    fun intent(): Intent = Intent(Intent.ACTION_VIEW)
        .setDataAndType(ContentUris.withAppendedId(ContactsContract.Data.CONTENT_URI, dataId), mimeType)
}

class ContactPhone(val number: String, val primary: Boolean, val mobile: Boolean)

/** A contact with what search can do with it. */
class ContactResult(
    val id: Long,
    val lookupKey: String,
    val name: String,
    val photo: ImageBitmap?,
    val phones: List<ContactPhone> = emptyList(),
    val emails: List<String> = emptyList(),
    val apps: List<ContactApp> = emptyList(),
) {
    val uri: Uri get() = ContactsContract.Contacts.getLookupUri(id, lookupKey)

    /** The number to call: the default one, else a mobile, else the first. */
    val phone: String? get() = (phones.firstOrNull { it.primary } ?: phones.firstOrNull { it.mobile } ?: phones.firstOrNull())?.number

    fun chatKey(app: ContactApp?) = "chat:$lookupKey:${app?.pkg ?: "sms"}"
    val callKey get() = "call:$lookupKey"
    fun videoKey(app: ContactApp) = "video:$lookupKey:${app.pkg}"

    fun app(kind: ContactApp.Kind, pkg: String? = null): ContactApp? {
        val list = apps.filter { it.kind == kind && (pkg == null || it.pkg == pkg) }
        return list.firstOrNull { it.pkg in WHATSAPP } ?: list.firstOrNull()
    }

    companion object {
        val WHATSAPP = setOf("com.whatsapp", "com.whatsapp.w4b")
    }
}

/** Something the search hands to another app: the query on the web, a number to call... */
class HandOff(val kind: Kind, val value: String) {
    enum class Kind { CALL, EMAIL, LINK, WEB, STORE, MAPS }

    fun intent(): Intent = when (kind) {
        Kind.CALL -> Intent(Intent.ACTION_DIAL, Uri.fromParts("tel", value, null))
        Kind.EMAIL -> Intent(Intent.ACTION_SENDTO, Uri.fromParts("mailto", value, null))
        Kind.LINK -> Intent(Intent.ACTION_VIEW, value.toUri())
        Kind.WEB -> Intent(Intent.ACTION_WEB_SEARCH).putExtra(SearchManager.QUERY, value)
        Kind.STORE -> Intent(Intent.ACTION_VIEW, ("market://search?q=" + Uri.encode(value)).toUri())
        Kind.MAPS -> Intent(Intent.ACTION_VIEW, ("geo:0,0?q=" + Uri.encode(value)).toUri())
    }
}

/**
 * What a command or a recognised query does ("Call Ana", "Message Ana · WhatsApp", "Start a 5
 * min timer"), shown above the apps.
 */
class QuickAction(
    val title: String,
    val subtitle: String?,
    @DrawableRes val icon: Int,
    val image: ImageBitmap? = null,
    val run: Run,
    /** What it is, for [ActionStats] ("chat:<contact>:com.whatsapp"); null = not learned. */
    val key: String? = null,
    /** The contact it is for, if any (their lookup key). */
    val person: String? = null,
) {
    sealed interface Run {
        class Open(val intent: Intent) : Run

        /** Places a phone call (asks for the permission the first time). */
        class Call(val number: String) : Run
        class Shortcut(val shortcut: ShortcutResult) : Run
        data object AllowContacts : Run
    }
}

/** Everything but apps for one query (apps are filtered by the drawer, on every key press). */
class SearchExtras(
    val query: String = "",
    /** What the query asks for (a command, a person, an address), above the apps. */
    val top: List<QuickAction> = emptyList(),
    /** The keyboard's Go key takes [top]'s first action before the apps. */
    val goFirst: Boolean = false,
    val calc: String? = null,
    val shortcuts: List<ShortcutResult> = emptyList(),
    val contacts: List<ContactResult> = emptyList(),
    val settings: List<SettingPage> = emptyList(),
    /** Call, email or open a link: what the query itself is. */
    val direct: List<HandOff> = emptyList(),
    /** Web, app store and maps for the query. */
    val handOffs: List<HandOff> = emptyList(),
) {
    val isEmpty get() = top.isEmpty() && calc == null && shortcuts.isEmpty() && contacts.isEmpty() && settings.isEmpty() && direct.isEmpty()
}
