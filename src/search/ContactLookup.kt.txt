package io.github.jacomartins.canyonui.search

import android.content.Context
import android.graphics.BitmapFactory
import android.net.Uri
import android.provider.ContactsContract
import android.provider.ContactsContract.CommonDataKinds
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.asImageBitmap
import androidx.core.net.toUri

/**
 * Contacts by name, with their numbers, email addresses and the ways other apps added to reach
 * them (WhatsApp, Telegram, Signal...). Needs the contacts permission; callers check it.
 */
class ContactLookup(context: Context, private val appLabels: () -> Map<String, String>) {
    private val resolver = context.applicationContext.contentResolver

    // Contact photos already decoded, by their address: typing decodes each one once.
    private val photos = android.util.LruCache<String, ImageBitmap>(PHOTO_CACHE)

    /** Up to [limit] contacts matching [name], best first. Throws SecurityException without permission. */
    fun find(name: String, limit: Int): List<ContactResult> {
        val uri = Uri.withAppendedPath(ContactsContract.Contacts.CONTENT_FILTER_URI, Uri.encode(name))
        val projection = arrayOf(
            ContactsContract.Contacts._ID,
            ContactsContract.Contacts.LOOKUP_KEY,
            ContactsContract.Contacts.DISPLAY_NAME_PRIMARY,
            ContactsContract.Contacts.PHOTO_THUMBNAIL_URI,
        )
        class Row(val id: Long, val lookup: String, val name: String, val photo: String?)
        val rows = ArrayList<Row>()
        try {
            resolver.query(uri, projection, null, null, null)?.use { c ->
                while (c.moveToNext() && rows.size < limit) {
                    rows += Row(c.getLong(0), c.getString(1) ?: continue, c.getString(2) ?: continue, c.getString(3))
                }
            }
        } catch (e: IllegalArgumentException) {
            return emptyList()
        }
        if (rows.isEmpty()) return emptyList()
        val details = details(rows.map { it.id })
        return rows.map { r ->
            val d = details[r.id]
            ContactResult(
                r.id, r.lookup, r.name, r.photo?.let { p -> photos.get(p) ?: photo(p.toUri())?.also { photos.put(p, it) } },
                d?.phones.orEmpty(), d?.emails.orEmpty(), d?.apps.orEmpty(),
            )
        }
    }

    private class Details {
        val phones = ArrayList<ContactPhone>()
        val emails = ArrayList<String>()
        val apps = ArrayList<ContactApp>()
    }

    private fun details(ids: List<Long>): Map<Long, Details> {
        val labels = appLabels()
        return try {
            details(ids, labels, withAccount = true)
        } catch (e: IllegalArgumentException) {
            // A contacts provider without the account column: the app comes from the entry's type.
            details(ids, labels, withAccount = false)
        }
    }

    private fun details(ids: List<Long>, labels: Map<String, String>, withAccount: Boolean): Map<Long, Details> {
        val out = HashMap<Long, Details>()
        val projection = listOfNotNull(
            ContactsContract.Data._ID,
            ContactsContract.Data.CONTACT_ID,
            ContactsContract.Data.MIMETYPE,
            ContactsContract.Data.DATA1,
            ContactsContract.Data.DATA2,
            ContactsContract.Data.IS_SUPER_PRIMARY,
            ContactsContract.RawContacts.ACCOUNT_TYPE.takeIf { withAccount },
        ).toTypedArray()
        val selection = "${ContactsContract.Data.CONTACT_ID} IN (${ids.joinToString(",")})"
        resolver.query(ContactsContract.Data.CONTENT_URI, projection, selection, null, null)?.use { c ->
            val seenApps = HashSet<String>()
            while (c.moveToNext()) {
                val d = out.getOrPut(c.getLong(1)) { Details() }
                val mime = c.getString(2) ?: continue
                when (mime) {
                    CommonDataKinds.Phone.CONTENT_ITEM_TYPE -> {
                        val number = c.getString(3) ?: continue
                        if (d.phones.none { same(it.number, number) }) {
                            d.phones += ContactPhone(number, c.getInt(5) != 0, c.getInt(4) == CommonDataKinds.Phone.TYPE_MOBILE)
                        }
                    }
                    CommonDataKinds.Email.CONTENT_ITEM_TYPE -> c.getString(3)?.let { if (it !in d.emails) d.emails += it }
                    else -> {
                        // An app's own entry: its account type is the app's package (and the
                        // type names it too: "vnd.com.whatsapp.profile").
                        if (!mime.startsWith("vnd.android.cursor.item/vnd.")) continue
                        val pkg = (if (withAccount) c.getString(6) else null)?.takeIf { it in labels }
                            ?: labels.keys.filter { it in mime }.maxByOrNull { it.length }
                            ?: continue
                        val label = labels[pkg] ?: continue
                        val m = mime.lowercase()
                        val kind = when {
                            "video" in m -> ContactApp.Kind.VIDEO
                            "call" in m || "voip" in m -> ContactApp.Kind.CALL
                            "profile" in m || "message" in m || "chat" in m -> ContactApp.Kind.CHAT
                            else -> continue
                        }
                        // One of each per app and contact (an app may add one per number).
                        if (seenApps.add("${c.getLong(1)}|$pkg|$kind")) d.apps += ContactApp(c.getLong(0), mime, pkg, label, kind)
                    }
                }
            }
        }
        return out
    }

    private fun same(a: String, b: String) = a.filter { it.isDigit() }.takeLast(9) == b.filter { it.isDigit() }.takeLast(9)

    private fun photo(uri: Uri): ImageBitmap? = try {
        resolver.openInputStream(uri)?.use { BitmapFactory.decodeStream(it) }?.asImageBitmap()
    } catch (e: Exception) {
        null
    }

    private companion object {
        const val PHOTO_CACHE = 48
    }
}
