package io.github.jacomartins.canyonui.data

import android.content.Context
import android.content.pm.ActivityInfo
import android.content.pm.LauncherActivityInfo
import android.content.pm.PackageManager
import android.content.res.Configuration
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import android.graphics.BitmapShader
import android.graphics.Canvas
import android.graphics.Color
import android.graphics.Paint
import android.graphics.Path
import android.graphics.PorterDuff
import android.graphics.PorterDuffColorFilter
import android.graphics.PorterDuffXfermode
import android.graphics.Rect
import android.graphics.Shader
import android.graphics.drawable.AdaptiveIconDrawable
import android.graphics.drawable.Drawable
import android.graphics.drawable.LayerDrawable
import android.os.Build
import android.os.Process
import android.util.Log
import android.util.LruCache
import androidx.core.graphics.createBitmap
import androidx.core.graphics.scale
import androidx.core.graphics.drawable.toBitmap
import androidx.core.graphics.drawable.toDrawable
import io.github.jacomartins.canyonui.model.AppCategories
import io.github.jacomartins.canyonui.model.AppKey
import java.io.File
import java.util.Calendar

/**
 * Rendered app icons, keyed by [AppKey], bounded by bytes. Where an icon comes from, first match:
 *
 * 1. The user's own choice for the app ([overrides]): an image, or an icon from any pack.
 * 2. The active icon [pack], when it covers the app.
 * 3. A dynamic clock, when the app declares one (the AOSP launcher's clock metadata).
 * 4. The app's own icon, drawn per [style]: as the phone hands it out ([IconShape.SYSTEM]), or
 *    rebuilt from its adaptive layers in the chosen shape, themed from its monochrome layer, or
 *    (legacy icons) set on a plate of the shape. Then the pack's style, if it has one.
 *
 * Rendering runs off the main thread except [tickClocks], which redraws clock icons in place.
 */
class IconCache(private val context: Context) {
    private val resources = context.resources
    val iconSizePx: Int = (ICON_DP * context.resources.displayMetrics.density).toInt()
    private val density = context.resources.displayMetrics.densityDpi

    private val cache = object : LruCache<AppKey, Bitmap>(MAX_BYTES) {
        override fun sizeOf(key: AppKey, value: Bitmap) = value.allocationByteCount
    }

    @Volatile
    var pack: IconPack? = null
        private set

    @Volatile
    var style = IconStyle()
        private set

    @Volatile
    private var overrides: Map<String, IconOverride> = emptyMap()

    /** Packs other than the active one that per-app choices point at. */
    private val otherPacks = HashMap<String, IconPack?>()
    private val clocks = HashMap<AppKey, ClockIcon>()

    /** Switches packs and drops every rendered icon. */
    fun setPack(newPack: IconPack?) {
        pack = newPack
        evictAll()
    }

    fun setStyle(newStyle: IconStyle, newOverrides: Map<String, IconOverride>) {
        val old = overrides
        val styleChanged = newStyle != style
        style = newStyle
        overrides = newOverrides
        if (styleChanged) {
            evictAll()
            return
        }
        // Only the apps whose own choice changed are drawn again.
        for (k in old.keys + newOverrides.keys) {
            if (old[k] == newOverrides[k]) continue
            val key = AppKey.parse(k) ?: continue
            cache.remove(key)
            synchronized(clocks) { clocks.remove(key) }
        }
    }

    private fun evictAll() {
        cache.evictAll()
        synchronized(clocks) { clocks.clear() }
    }

    fun labelOverride(key: AppKey): String? = overrides[key.toString()]?.label?.takeIf { it.isNotBlank() }

    fun categoryOverride(key: AppKey): AppCategories.Category? = AppCategories.Category.fromKey(overrides[key.toString()]?.category)

    /** Themed icons follow light/dark mode and the wallpaper palette. */
    fun onConfigurationChanged() {
        if (style.themed) evictAll()
    }

    /** Calendar icons from the pack change with the date. */
    fun invalidateCalendars() {
        val p = pack ?: return
        cache.snapshot().keys.filter { p.isCalendar(it.packageName, it.className) }.forEach { cache.remove(it) }
    }

    fun invalidatePackage(packageName: String) {
        cache.snapshot().keys.filter { it.packageName == packageName }.forEach { cache.remove(it) }
        synchronized(clocks) { clocks.keys.removeAll { it.packageName == packageName } }
        synchronized(otherPacks) { otherPacks.remove(packageName) }
    }

    fun isClock(key: AppKey) = synchronized(clocks) { key in clocks }

    fun get(info: LauncherActivityInfo, key: AppKey, pm: PackageManager): Bitmap =
        cache.get(key) ?: render(info, key, pm).also { cache.put(key, it) }

    private fun render(info: LauncherActivityInfo, key: AppKey, pm: PackageManager): Bitmap {
        val bmp = try {
            chosenIcon(key)
                ?: pack?.drawableFor(key.packageName, key.className, density)?.let(::fromPack)
                ?: clockIcon(info, key, pm)
                ?: appIcon(info, key, pm)
        } catch (e: RuntimeException) {
            // A broken app or pack resource must not take the app list down.
            Log.w(TAG, "Icon for $key failed; using the system icon", e)
            info.getIcon(density).toBitmap(iconSizePx, iconSizePx, Bitmap.Config.ARGB_8888)
        }
        if (info.user == Process.myUserHandle()) return bmp
        val badged = pm.getUserBadgedIcon(bmp.toDrawable(resources), info.user)
        return badged.toBitmap(iconSizePx, iconSizePx, Bitmap.Config.ARGB_8888)
    }

    // ------------------------------------------------------------------------------- sources

    private fun chosenIcon(key: AppKey): Bitmap? {
        val o = overrides[key.toString()] ?: return null
        o.file?.let { name ->
            val src = BitmapFactory.decodeFile(File(IconFiles.dir(context), name).path) ?: return@let
            return shaped(src)
        }
        val pkg = o.packPackage ?: return null
        val name = o.drawable ?: return null
        val p = if (pkg == pack?.packageName) pack else synchronized(otherPacks) { otherPacks.getOrPut(pkg) { IconPack.load(context, pkg) } }
        return p?.drawableNamed(name, density)?.let(::fromPack)
    }

    /** Pack artwork is used as drawn (its own shape); adaptive pack icons get the chosen shape. */
    private fun fromPack(d: Drawable): Bitmap =
        if (d is AdaptiveIconDrawable) adaptive(d) else d.toBitmap(iconSizePx, iconSizePx, Bitmap.Config.ARGB_8888)

    private fun appIcon(info: LauncherActivityInfo, key: AppKey, pm: PackageManager): Bitmap {
        val s = style
        val base = if (s.shape == IconShape.SYSTEM && !s.themed) {
            val d = info.getIcon(density)
            if (d is AdaptiveIconDrawable) adaptive(d) else d.toBitmap(iconSizePx, iconSizePx, Bitmap.Config.ARGB_8888)
        } else {
            // The app's own resources: the icon before a device theme reworks it (shadows, plates).
            val raw = rawIcon(info, pm) ?: info.getIcon(density)
            val mono = if (Build.VERSION.SDK_INT >= 33 && s.themed) (raw as? AdaptiveIconDrawable)?.monochrome else null
            when {
                mono != null && Build.VERSION.SDK_INT >= 33 -> themed(mono)
                raw is AdaptiveIconDrawable -> adaptive(raw)
                else -> onPlate(raw)
            }
        }
        val ps = pack?.style
        return if (ps != null && s.packStyle) packStyled(base, ps, key) else base
    }

    private fun rawIcon(info: LauncherActivityInfo, pm: PackageManager): Drawable? = try {
        val ai: ActivityInfo = if (Build.VERSION.SDK_INT >= 31) info.activityInfo else pm.getActivityInfo(info.componentName, 0)
        val id = ai.iconResource
        if (id == 0) null else pm.getResourcesForApplication(info.applicationInfo).getDrawableForDensity(id, density, null)
    } catch (e: Exception) {
        null
    }

    // ------------------------------------------------------------------------------- drawing

    private val shapePath = Path()
    private val aa = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG)

    /** The outline of the chosen shape, or the phone's own adaptive icon mask. */
    private fun outline(size: Int): Path {
        style.shape.path(size.toFloat(), shapePath)?.let { return it }
        val probe = AdaptiveIconDrawable(Color.TRANSPARENT.toDrawable(), Color.TRANSPARENT.toDrawable())
        probe.setBounds(0, 0, size, size)
        return Path(probe.iconMask)
    }

    /** Fills the outline with [content], anti-aliased (a clip on a bitmap canvas is not). */
    private fun shaped(content: Bitmap, out: Bitmap? = null): Bitmap {
        val size = iconSizePx
        val target = out ?: createBitmap(size, size)
        val c = Canvas(target)
        if (out != null) c.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR)
        val shader = BitmapShader(content, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP)
        if (content.width != size || content.height != size) {
            // Centre-crop the source into the square.
            val scale = size.toFloat() / minOf(content.width, content.height)
            val m = android.graphics.Matrix()
            m.setScale(scale, scale)
            m.postTranslate((size - content.width * scale) / 2f, (size - content.height * scale) / 2f)
            shader.setLocalMatrix(m)
        }
        val p = Paint(aa).apply { this.shader = shader }
        synchronized(shapePath) { c.drawPath(outline(size), p) }
        return target
    }

    /**
     * An adaptive icon from its own layers (1.5x the visible size, centred, per the spec), so
     * effects a device wraps around the drawable (such as a drop shadow) are left out.
     */
    private fun adaptive(d: AdaptiveIconDrawable, out: Bitmap? = null): Bitmap {
        val size = iconSizePx
        val layers = createBitmap(size, size)
        val c = Canvas(layers)
        val inset = size / 4
        for (layer in listOf(d.background, d.foreground)) {
            layer ?: continue
            layer.setBounds(-inset, -inset, size + inset, size + inset)
            layer.draw(c)
        }
        return shaped(layers, out).also { layers.recycle() }
    }

    /** Android 13 themed icon: the monochrome layer in the palette's accent on a soft plate. */
    @androidx.annotation.RequiresApi(33)
    private fun themed(mono: Drawable): Bitmap {
        val size = iconSizePx
        val night = resources.configuration.uiMode and Configuration.UI_MODE_NIGHT_MASK == Configuration.UI_MODE_NIGHT_YES
        // A tinted glyph on a pale plate of the same hue (Android 13's own pair draws the glyph
        // in a grey that reads as washed out over a colourful wallpaper).
        val bg = context.getColor(if (night) android.R.color.system_neutral1_800 else android.R.color.system_accent1_100)
        val fg = context.getColor(if (night) android.R.color.system_accent1_200 else android.R.color.system_accent1_700)
        val layers = createBitmap(size, size)
        val c = Canvas(layers)
        c.drawColor(bg)
        val inset = size / 4
        val m = mono.mutate()
        m.colorFilter = PorterDuffColorFilter(fg, PorterDuff.Mode.SRC_IN)
        m.setBounds(-inset, -inset, size + inset, size + inset)
        m.draw(c)
        return shaped(layers).also { layers.recycle() }
    }

    /** A legacy (non-adaptive) icon, centred at 70% on a white plate of the chosen shape. */
    private fun onPlate(d: Drawable): Bitmap {
        val size = iconSizePx
        val layers = createBitmap(size, size)
        val c = Canvas(layers)
        c.drawColor(Color.WHITE)
        val inner = (size * 0.7f).toInt()
        val off = (size - inner) / 2
        d.toBitmap(inner, inner, Bitmap.Config.ARGB_8888).let { c.drawBitmap(it, off.toFloat(), off.toFloat(), aa) }
        return shaped(layers).also { layers.recycle() }
    }

    /** The pack's back plate, the icon scaled and cut by the pack's mask, then its overlay. */
    private fun packStyled(base: Bitmap, ps: IconPack.PackStyle, key: AppKey): Bitmap {
        val p = pack ?: return base
        val size = iconSizePx
        val full = Rect(0, 0, size, size)
        val out = createBitmap(size, size)
        val c = Canvas(out)
        // One of the pack's plates, the same one for an app every time.
        ps.back.takeIf { it.isNotEmpty() }?.let { backs ->
            p.drawableNamed(backs[Math.floorMod(key.packageName.hashCode(), backs.size)], density)?.apply { bounds = full }?.draw(c)
        }
        val layer = createBitmap(size, size)
        val lc = Canvas(layer)
        val inner = (size * ps.scale).toInt()
        val off = (size - inner) / 2f
        lc.drawBitmap(base, null, android.graphics.RectF(off, off, off + inner, off + inner), aa)
        ps.mask?.let { name ->
            p.drawableNamed(name, density)?.let { m ->
                val maskBmp = m.toBitmap(size, size, Bitmap.Config.ARGB_8888)
                lc.drawBitmap(maskBmp, 0f, 0f, Paint().apply { xfermode = PorterDuffXfermode(PorterDuff.Mode.DST_OUT) })
            }
        }
        c.drawBitmap(layer, 0f, 0f, null)
        layer.recycle()
        ps.upon?.let { name -> p.drawableNamed(name, density)?.apply { bounds = full }?.draw(c) }
        return out
    }

    // ------------------------------------------------------------------------------- clocks

    /**
     * A clock icon whose hands are layers the app marks in its manifest (the metadata the AOSP
     * launcher reads). Its bitmap is redrawn in place every minute by [tickClocks].
     */
    private class ClockIcon(
        val drawable: AdaptiveIconDrawable,
        val hands: LayerDrawable,
        val hour: Int,
        val minute: Int,
        val second: Int,
        val defaultHour: Int,
        val defaultMinute: Int,
        val defaultSecond: Int,
        val bitmap: Bitmap,
    )

    private fun clockIcon(info: LauncherActivityInfo, key: AppKey, pm: PackageManager): Bitmap? {
        // A themed icon has no hands to move.
        if (style.themed) return null
        val meta = try {
            pm.getApplicationInfo(key.packageName, PackageManager.GET_META_DATA).metaData
        } catch (e: PackageManager.NameNotFoundException) {
            null
        } ?: return null
        val resId = meta.getInt("$META.LEVEL_PER_TICK_ICON_ROUND", 0)
        if (resId == 0) return null
        val d = try {
            pm.getResourcesForApplication(info.applicationInfo).getDrawableForDensity(resId, density, null)
        } catch (e: Exception) {
            null
        }?.mutate() as? AdaptiveIconDrawable ?: return null
        val hands = d.foreground as? LayerDrawable ?: return null
        val clock = ClockIcon(
            d, hands,
            meta.getInt("$META.HOUR_LAYER_INDEX", -1),
            meta.getInt("$META.MINUTE_LAYER_INDEX", -1),
            meta.getInt("$META.SECOND_LAYER_INDEX", -1),
            meta.getInt("$META.DEFAULT_HOUR", 0),
            meta.getInt("$META.DEFAULT_MINUTE", 0),
            meta.getInt("$META.DEFAULT_SECOND", 0),
            createBitmap(iconSizePx, iconSizePx),
        )
        drawClock(clock)
        synchronized(clocks) { clocks[key] = clock }
        return clock.bitmap
    }

    /** Sets the hands to now (the AOSP level scheme) and redraws into the clock's own bitmap. */
    private fun drawClock(c: ClockIcon) {
        val t = Calendar.getInstance()
        val hour = (t.get(Calendar.HOUR) + (12 - c.defaultHour)) % 12
        val minute = (t.get(Calendar.MINUTE) + (60 - c.defaultMinute)) % 60
        val second = (t.get(Calendar.SECOND) + (60 - c.defaultSecond)) % 60
        if (c.hour in 0 until c.hands.numberOfLayers) c.hands.getDrawable(c.hour).level = hour * 60 + t.get(Calendar.MINUTE)
        if (c.minute in 0 until c.hands.numberOfLayers) c.hands.getDrawable(c.minute).level = minute + t.get(Calendar.HOUR) * 60
        if (c.second in 0 until c.hands.numberOfLayers) c.hands.getDrawable(c.second).level = second * 10
        adaptive(c.drawable, c.bitmap)
    }

    /** Redraws every clock icon for the current minute. Returns true if there are any. */
    fun tickClocks(): Boolean {
        val all = synchronized(clocks) { clocks.values.toList() }
        all.forEach(::drawClock)
        return all.isNotEmpty()
    }

    companion object {
        /** Largest on-screen size (IconSize.LARGE cap), so icons are only ever scaled down. */
        const val ICON_DP = 80
        private const val MAX_BYTES = 24 * 1024 * 1024
        private const val TAG = "IconCache"
        private const val META = "com.android.launcher3"
    }
}

/** Images the user picked as app icons, kept in the app's private storage. */
object IconFiles {
    fun dir(context: Context) = File(context.filesDir, "icons").apply { mkdirs() }

    /** Copies [uri] (from the photo picker) as a square PNG; returns its file name. */
    fun save(context: Context, uri: android.net.Uri): String? = try {
        val src = context.contentResolver.openInputStream(uri)?.use { BitmapFactory.decodeStream(it) } ?: return null
        val side = minOf(src.width, src.height)
        val square = Bitmap.createBitmap(src, (src.width - side) / 2, (src.height - side) / 2, side, side)
        val scaled = square.scale(SIZE, SIZE)
        val name = "icon_${System.currentTimeMillis()}.png"
        File(dir(context), name).outputStream().use { scaled.compress(Bitmap.CompressFormat.PNG, 100, it) }
        name
    } catch (e: Exception) {
        null
    }

    fun delete(context: Context, name: String) {
        File(dir(context), name).delete()
    }

    private const val SIZE = 256
}
