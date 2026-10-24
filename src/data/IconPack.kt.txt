package io.github.jacomartins.canyonui.data

import android.annotation.SuppressLint
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.content.res.Resources
import android.graphics.drawable.Drawable
import android.util.Log
import android.util.Xml
import io.github.jacomartins.canyonui.model.AppFilter
import org.xmlpull.v1.XmlPullParser
import java.util.Calendar

/** An installed icon pack, for the chooser. */
data class IconPackInfo(val packageName: String, val label: String)

/**
 * A loaded icon pack: the parsed `appfilter.xml` plus the pack's resources. Drawable names are
 * resolved to ids lazily and remembered. Thread-safe for concurrent reads after construction;
 * the id cache is guarded by its own lock.
 */
class IconPack private constructor(
    val packageName: String,
    private val res: Resources,
    private val byComponent: Map<Pair<String, String>, String>,
    private val byPackage: Map<String, String>,
    private val calendarByComponent: Map<Pair<String, String>, String>,
    private val calendarByPackage: Map<String, String>,
    /** The pack's style for apps it does not cover; null when it defines none. */
    val style: PackStyle?,
) {
    /**
     * Drawn under, through and over an icon the pack does not cover: a back plate, a mask that
     * cuts away (opaque = removed), an overlay, and the scale the original icon is drawn at.
     */
    class PackStyle(val back: List<String>, val mask: String?, val upon: String?, val scale: Float)

    private val ids = HashMap<String, Int>()

    val entryCount get() = byComponent.size

    /** True when this pack draws a date-dependent icon for the app. */
    fun isCalendar(pkg: String, cls: String) = AppFilter.lookup(pkg, cls, calendarByComponent, calendarByPackage) != null

    /** The pack's icon for an activity, or null when the pack does not cover it. */
    fun drawableFor(pkg: String, cls: String, densityDpi: Int): Drawable? {
        val id = calendarId(pkg, cls)
            ?: AppFilter.lookup(pkg, cls, byComponent, byPackage)?.let(::idFor)
            ?: return null
        if (id == 0) return null
        return try {
            res.getDrawableForDensity(id, densityDpi, null)
        } catch (e: Resources.NotFoundException) {
            null
        } catch (e: OutOfMemoryError) {
            null
        }
    }

    /** A drawable of the pack by name (style parts, the per-app icon chooser). */
    fun drawableNamed(name: String, densityDpi: Int): Drawable? {
        val id = idFor(name)
        if (id == 0) return null
        return try {
            res.getDrawableForDensity(id, densityDpi, null)
        } catch (e: Resources.NotFoundException) {
            null
        } catch (e: OutOfMemoryError) {
            null
        }
    }

    /**
     * Every icon the pack offers, for choosing one by hand: its drawable.xml list when it has
     * one (in res/xml or assets), else every icon its appfilter maps.
     */
    @SuppressLint("DiscouragedApi")
    fun allDrawables(context: Context): List<String> {
        val parser: XmlPullParser? = res.getIdentifier("drawable", "xml", packageName).takeIf { it != 0 }?.let { res.getXml(it) }
            ?: try {
                val bytes = context.createPackageContext(packageName, 0).assets.open("drawable.xml").use { it.readBytes() }
                Xml.newPullParser().apply { setInput(bytes.inputStream(), null) }
            } catch (e: Exception) {
                null
            }
        val out = LinkedHashSet<String>()
        if (parser != null) {
            try {
                var event = parser.eventType
                while (event != XmlPullParser.END_DOCUMENT) {
                    if (event == XmlPullParser.START_TAG && parser.name == "item") {
                        parser.getAttributeValue(null, "drawable")?.takeIf { it.isNotBlank() }?.let(out::add)
                    }
                    event = parser.next()
                }
            } catch (e: Exception) {
                Log.w(TAG, "drawable.xml of $packageName is malformed after ${out.size} entries", e)
            }
        }
        if (out.isEmpty()) out += byComponent.values
        return out.toList()
    }

    /** Today's calendar icon: the weekday variant if the pack has it, else the date-only one. */
    private fun calendarId(pkg: String, cls: String): Int? {
        val prefix = AppFilter.lookup(pkg, cls, calendarByComponent, calendarByPackage) ?: return null
        val now = Calendar.getInstance()
        val day = now.get(Calendar.DAY_OF_MONTH)
        val withWeekday = idFor(AppFilter.calendarWeekdayDrawable(prefix, day, now.get(Calendar.DAY_OF_WEEK)))
        val id = if (withWeekday != 0) withWeekday else idFor(AppFilter.calendarDrawable(prefix, day))
        return id.takeIf { it != 0 }
    }

    // Icon pack resources can only be found by name; there is no compile-time R class for them.
    @SuppressLint("DiscouragedApi")
    private fun idFor(name: String): Int = synchronized(ids) {
        ids.getOrPut(name) { res.getIdentifier(name, "drawable", packageName) }
    }

    companion object {
        private const val TAG = "IconPack"

        /** Intent actions icon packs declare for the common launcher conventions. */
        private val ACTIONS = listOf(
            "org.adw.launcher.THEMES",
            "com.novalauncher.THEME",
            "com.gau.go.launcherex.theme",
            "com.anddoes.launcher.THEME",
            "com.teslacoilsw.launcher.THEME",
        )

        fun installed(context: Context): List<IconPackInfo> {
            val pm = context.packageManager
            val pkgs = LinkedHashSet<String>()
            for (action in ACTIONS) {
                queryActivities(pm, Intent(action)).forEach { pkgs += it.activityInfo.packageName }
            }
            return pkgs.mapNotNull { pkg ->
                try {
                    val ai = pm.getApplicationInfo(pkg, 0)
                    IconPackInfo(pkg, pm.getApplicationLabel(ai).toString())
                } catch (e: PackageManager.NameNotFoundException) {
                    null
                }
            }.sortedBy { it.label.lowercase() }
        }

        /** Parses the pack. Returns null if it is not installed or has no readable appfilter. */
        fun load(context: Context, packageName: String): IconPack? {
            val res = try {
                context.packageManager.getResourcesForApplication(packageName)
            } catch (e: PackageManager.NameNotFoundException) {
                return null
            }
            val parser = openAppFilter(context, res, packageName) ?: return null
            val byComponent = HashMap<Pair<String, String>, String>()
            val byPackage = HashMap<String, String>()
            val calByComponent = HashMap<Pair<String, String>, String>()
            val calByPackage = HashMap<String, String>()
            val backs = ArrayList<String>()
            var mask: String? = null
            var upon: String? = null
            var scale = 1f
            try {
                var event = parser.eventType
                while (event != XmlPullParser.END_DOCUMENT) {
                    if (event == XmlPullParser.START_TAG) {
                        val component = AppFilter.parseComponent(parser.getAttributeValue(null, "component"))
                        when (parser.name) {
                            "item" -> {
                                val drawable = parser.getAttributeValue(null, "drawable")
                                if (component != null && !drawable.isNullOrBlank()) {
                                    byComponent.putIfAbsent(component, drawable)
                                    byPackage.putIfAbsent(component.first, drawable)
                                }
                            }
                            // Style for icons the pack does not cover (img1, img2, ... attributes).
                            "iconback" -> for (i in 0 until parser.attributeCount) {
                                if (parser.getAttributeName(i).startsWith("img")) parser.getAttributeValue(i).takeIf { it.isNotBlank() }?.let(backs::add)
                            }
                            "iconmask" -> mask = parser.getAttributeValue(null, "img1")?.takeIf { it.isNotBlank() }
                            "iconupon" -> upon = parser.getAttributeValue(null, "img1")?.takeIf { it.isNotBlank() }
                            "scale" -> scale = parser.getAttributeValue(null, "factor")?.toFloatOrNull()?.coerceIn(0.1f, 2f) ?: 1f
                            "calendar" -> {
                                val prefix = parser.getAttributeValue(null, "prefix")
                                if (component != null && !prefix.isNullOrBlank()) {
                                    calByComponent.putIfAbsent(component, prefix)
                                    calByPackage.putIfAbsent(component.first, prefix)
                                }
                            }
                        }
                    }
                    event = parser.next()
                }
            } catch (e: Exception) {
                // A broken pack must never take the launcher down; keep what was read.
                Log.w(TAG, "appfilter of $packageName is malformed after ${byComponent.size} entries", e)
            }
            Log.i(TAG, "Loaded $packageName: ${byComponent.size} icons, ${calByComponent.size} calendars")
            val style = if (backs.isEmpty() && mask == null && upon == null) null else PackStyle(backs, mask, upon, scale)
            return IconPack(packageName, res, byComponent, byPackage, calByComponent, calByPackage, style)
        }

        @SuppressLint("DiscouragedApi") // appfilter is looked up by name inside another package.
        private fun openAppFilter(context: Context, res: Resources, pkg: String): XmlPullParser? {
            val xmlId = res.getIdentifier("appfilter", "xml", pkg)
            if (xmlId != 0) return res.getXml(xmlId)
            return try {
                val bytes = context.createPackageContext(pkg, 0).assets.open("appfilter.xml").use { it.readBytes() }
                Xml.newPullParser().apply { setInput(bytes.inputStream(), null) }
            } catch (e: Exception) {
                Log.w(TAG, "No appfilter in $pkg", e)
                null
            }
        }

        private fun queryActivities(pm: PackageManager, intent: Intent) =
            if (android.os.Build.VERSION.SDK_INT >= 33) {
                pm.queryIntentActivities(intent, PackageManager.ResolveInfoFlags.of(0))
            } else {
                @Suppress("DEPRECATION")
                pm.queryIntentActivities(intent, 0)
            }
    }
}
