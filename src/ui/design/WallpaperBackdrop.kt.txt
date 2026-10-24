package io.github.jacomartins.canyonui.ui.design

import android.app.WallpaperManager
import android.content.Context
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.ColorMatrix
import android.graphics.ColorMatrixColorFilter
import android.graphics.ImageDecoder
import android.graphics.Paint
import android.graphics.Path
import android.graphics.Rect
import android.graphics.RectF
import android.net.Uri
import android.os.Build
import android.os.Handler
import android.os.Looper
import android.os.SystemClock
import android.util.Log
import android.view.View
import android.view.ViewTreeObserver
import android.view.WindowManager
import androidx.core.content.edit
import androidx.core.graphics.createBitmap
import androidx.core.graphics.scale
import androidx.core.graphics.withClip
import io.github.jacomartins.canyonui.data.RenderMode
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import java.io.File
import kotlin.math.max
import kotlin.math.roundToInt

/**
 * The blurred wallpaper behind see-through surfaces in the frosted and glass modes.
 *
 * Android does not let ordinary apps read the wallpaper (that needs a system permission), so the
 * launcher keeps its own copy: the user picks a photo in settings, the launcher sets it as the
 * home wallpaper and keeps the image, or the user captures the screen once (with Android's
 * screen capture consent) while home shows only the wallpaper. When the wallpaper is changed
 * anywhere else the copy no longer matches what is on screen, so it is dropped.
 *
 * Without a copy (a wallpaper set elsewhere, a solid colour, a live wallpaper) the effects use
 * the wallpaper's colours, which Android does share: exact for a solid colour, a gradient of the
 * main colours otherwise.
 */
object WallpaperBackdrop {

    /** A blurred, screen-sized wallpaper (stored small; draw it stretched to [screenW] x [screenH]). */
    class Blur(val bitmap: Bitmap, val screenW: Int, val screenH: Int, val mode: RenderMode)

    private const val TAG = "WallpaperBackdrop"
    private const val FILE = "wallpaper_copy.png"
    private const val PREFS = "wallpaper_copy"
    private const val KEY_VALID = "valid"
    private const val KEY_ID = "wallpaper_id"
    private const val KEY_LIVE = "live_component"

    /** A wallpaper change this soon after setting our own is our own. */
    private const val OWN_CHANGE_MS = 15_000L

    private val main = Handler(Looper.getMainLooper())
    private var mode = RenderMode.TRANSLUCENT
    private var copy: Bitmap? = null
    private var setAt = 0L
    private var listening = false

    private val _hasCopy = MutableStateFlow(false)

    /** The effects use the wallpaper's colours only (no copy of the image). */
    private val _colorsOnly = MutableStateFlow(false)
    val colorsOnly: StateFlow<Boolean> = _colorsOnly.asStateFlow()

    /** True while the launcher's copy matches the wallpaper on screen. */
    val hasCopy: StateFlow<Boolean> = _hasCopy.asStateFlow()

    private val _source = MutableStateFlow<Bitmap?>(null)

    /** The sharp wallpaper copy while a blur mode is on (what Haze blurs itself); else null. */
    val source: StateFlow<Bitmap?> = _source.asStateFlow()

    private val _active = MutableStateFlow<Blur?>(null)

    /** The blur to draw behind surfaces now; null = plain translucent surfaces. */
    val active: StateFlow<Blur?> = _active.asStateFlow()

    /** Called with the settings' mode; loads the copy the first time it is needed. */
    suspend fun setMode(context: Context, newMode: RenderMode) {
        mode = newMode
        listen(context)
        if (copy == null && _hasCopy.value.not()) load(context)
        rebuild()
    }

    /**
     * Makes sure the blur shows what was last set here: reloads the kept copy from disk when it
     * is valid but not (or no longer) in memory, and rebuilds the blur. Home calls it when it
     * comes back, and it runs after a wallpaper is set.
     */
    suspend fun refresh(context: Context) {
        if (copy == null) load(context)
        if (copy != null && (_active.value == null || _active.value?.mode != mode)) rebuild()
    }

    /** True while [setFromPhoto] runs: the wallpaper changes under it, but the copy is coming. */
    @Volatile
    var setting = false
        private set

    /** Checks the copy still matches the wallpaper (when Android tells apps the wallpaper id). */
    suspend fun check(context: Context) {
        // Our own change: the id is saved once the set finishes (checking before that read the
        // previous id and threw the new copy away).
        if (setting || SystemClock.elapsedRealtime() - setAt < OWN_CHANGE_MS) return
        when {
            _hasCopy.value -> if (!stillSame(context)) invalidate(context) else return
            // Colours may have changed while away.
            _colorsOnly.value || copy == null -> fromColors(context)
            else -> return
        }
        rebuild()
    }

    /**
     * Makes [uri] (a photo the user picked) the wallpaper of [which] (home, lock or both:
     * WallpaperManager flags), cropped to the screen. A home wallpaper is kept as the copy to
     * blur. Returns false when it could not be read or set.
     */
    suspend fun setFromPhoto(
        context: Context,
        uri: Uri,
        which: Int = WallpaperManager.FLAG_SYSTEM,
        /** The part of the photo to use (0..1 of its width and height); null = its centre. */
        crop: RectF? = null,
    ): Boolean = withContext(Dispatchers.IO) {
        setting = true
        try {
            val (w, h) = screenSize(context)
            val source = ImageDecoder.createSource(context.contentResolver, uri)
            val decoded = ImageDecoder.decodeBitmap(source) { decoder, info, _ ->
                decoder.allocator = ImageDecoder.ALLOCATOR_SOFTWARE
                // Decode near the screen size (for the chosen part): enough detail, far less
                // memory than a full photo.
                val cw = crop?.width()?.coerceIn(0.01f, 1f) ?: 1f
                val ch = crop?.height()?.coerceIn(0.01f, 1f) ?: 1f
                val s = max(w / (info.size.width * cw), h / (info.size.height * ch)).coerceAtMost(1f)
                decoder.setTargetSize((info.size.width * s).roundToInt().coerceAtLeast(1), (info.size.height * s).roundToInt().coerceAtLeast(1))
            }
            val cropped = if (crop == null) centerCrop(decoded, w, h) else region(decoded, crop, w, h)
            val wm = WallpaperManager.getInstance(context)
            // Screen-sized wallpaper, so what is on screen is exactly the copy (no scrolling).
            runCatching { wm.suggestDesiredDimensions(w, h) }
            setAt = SystemClock.elapsedRealtime()
            val id = wm.setBitmap(cropped, null, true, which)
            if (which and WallpaperManager.FLAG_SYSTEM == 0) return@withContext true
            File(context.filesDir, FILE).outputStream().use { cropped.compress(Bitmap.CompressFormat.PNG, 100, it) }
            context.getSharedPreferences(PREFS, Context.MODE_PRIVATE).edit {
                putBoolean(KEY_VALID, true)
                putInt(KEY_ID, wallpaperId(context) ?: id)
                remove(KEY_LIVE)
            }
            copy = cropped
            _hasCopy.value = true
            _colorsOnly.value = false
            rebuild()
            true
        } catch (e: Exception) {
            Log.w(TAG, "Could not set the wallpaper", e)
            false
        } finally {
            setting = false
        }
    }

    private suspend fun load(context: Context) = withContext(Dispatchers.IO) {
        val prefs = context.getSharedPreferences(PREFS, Context.MODE_PRIVATE)
        val file = File(context.filesDir, FILE)
        if (prefs.getBoolean(KEY_VALID, false) && file.exists()) {
            copy = try {
                android.graphics.BitmapFactory.decodeFile(file.path)
            } catch (e: OutOfMemoryError) {
                null
            }
        }
        _hasCopy.value = copy != null
        if (copy == null) fromColors(context)
    }

    /** No copy of the image: one painted from the wallpaper's colours, when Android gives them. */
    private fun fromColors(context: Context) {
        val colors = try {
            WallpaperManager.getInstance(context).getWallpaperColors(WallpaperManager.FLAG_SYSTEM)
        } catch (e: Exception) {
            null
        }
        if (colors == null) {
            copy = null
            _colorsOnly.value = false
            return
        }
        val (w, h) = screenSize(context)
        val bmp = createBitmap(w.coerceAtLeast(1), h.coerceAtLeast(1))
        val c = Canvas(bmp)
        val main = colors.primaryColor.toArgb()
        val others = listOfNotNull(colors.secondaryColor?.toArgb(), colors.tertiaryColor?.toArgb())
        if (others.isEmpty()) {
            // A solid colour wallpaper: exactly that colour.
            c.drawColor(main)
        } else {
            val stops = (listOf(main) + others).toIntArray()
            c.drawPaint(Paint().apply { shader = android.graphics.LinearGradient(0f, 0f, w * 0.3f, h.toFloat(), stops, null, android.graphics.Shader.TileMode.CLAMP) })
        }
        copy = bmp
        _colorsOnly.value = true
    }

    private fun invalidate(context: Context) {
        context.getSharedPreferences(PREFS, Context.MODE_PRIVATE).edit { putBoolean(KEY_VALID, false) }
        File(context.filesDir, FILE).delete()
        copy = null
        _hasCopy.value = false
        _active.value = null
        _source.value = null
        fromColors(context)
    }

    /** The wallpaper's colours changed: still the copied wallpaper (a live one moving), or another one. */
    private fun stillSame(context: Context): Boolean {
        val prefs = context.getSharedPreferences(PREFS, Context.MODE_PRIVATE)
        val live = prefs.getString(KEY_LIVE, null)
        if (live != null) return WallpaperManager.getInstance(context).wallpaperInfo?.component?.flattenToString() == live
        val saved = prefs.getInt(KEY_ID, -1)
        // Android does not always tell the id: then it cannot be told apart, so it is kept.
        val now = wallpaperId(context) ?: return true
        return saved == -1 || now == saved
    }

    /** The wallpaper changing elsewhere (not by us, just now) makes the copy stale. */
    private fun listen(context: Context) {
        if (listening) return
        listening = true
        val app = context.applicationContext
        WallpaperManager.getInstance(app).addOnColorsChangedListener({ _, which ->
            if (which and WallpaperManager.FLAG_SYSTEM == 0) return@addOnColorsChangedListener
            if (setting || SystemClock.elapsedRealtime() - setAt <= OWN_CHANGE_MS) return@addOnColorsChangedListener
            when {
                // Painted from the colours: paint it again.
                _colorsOnly.value -> fromColors(app)
                _hasCopy.value && stillSame(app) -> return@addOnColorsChangedListener
                else -> invalidate(app)
            }
            kotlinx.coroutines.CoroutineScope(Dispatchers.Main).launch { rebuild() }
        }, main)
    }

    /**
     * Keeps [shot] (the screen captured while home showed only the wallpaper) as the copy. A live
     * wallpaper stays this one frame; it is kept until another wallpaper is set.
     */
    suspend fun setFromCapture(context: Context, shot: Bitmap) = withContext(Dispatchers.IO) {
        val app = context.applicationContext
        val (w, h) = screenSize(app)
        val img = if (shot.width == w && shot.height == h) shot else centerCrop(shot, w, h)
        File(app.filesDir, FILE).outputStream().use { img.compress(Bitmap.CompressFormat.PNG, 100, it) }
        val wm = WallpaperManager.getInstance(app)
        app.getSharedPreferences(PREFS, Context.MODE_PRIVATE).edit {
            putBoolean(KEY_VALID, true)
            putInt(KEY_ID, wallpaperId(app) ?: -1)
            val live = wm.wallpaperInfo?.component?.flattenToString()
            if (live != null) putString(KEY_LIVE, live) else remove(KEY_LIVE)
        }
        copy = img
        _hasCopy.value = true
        _colorsOnly.value = false
        rebuild()
    }

    private suspend fun rebuild() {
        val src = copy
        val m = mode
        if (src == null || m == RenderMode.TRANSLUCENT) {
            _active.value = null
            _source.value = null
            return
        }
        _source.value = src
        _active.value = withContext(Dispatchers.Default) { Blur(blur(src, m), src.width, src.height, m) }
    }

    /**
     * Blurs [src] cheaply: shrink it (most of the blur), then three box-blur passes to smooth the
     * shrunk pixels. Frosted shrinks more and lifts saturation a little, like frosted glass;
     * glass keeps more detail (half the screen's resolution: smooth when stretched, far from
     * full size).
     */
    private fun blur(src: Bitmap, mode: RenderMode): Bitmap {
        // Glass: half the screen's resolution and a light blur (the lens does the rest).
        val factor = if (mode == RenderMode.FROSTED) 12 else 2
        val radius = if (mode == RenderMode.FROSTED) 3 else 2
        val w = (src.width / factor).coerceAtLeast(1)
        val h = (src.height / factor).coerceAtLeast(1)
        val small = createBitmap(w, h)
        Canvas(small).drawBitmap(
            src, null, Rect(0, 0, w, h),
            Paint(Paint.FILTER_BITMAP_FLAG).apply {
                colorFilter = ColorMatrixColorFilter(ColorMatrix().apply { setSaturation(if (mode == RenderMode.FROSTED) 1.25f else 1.1f) })
            },
        )
        val px = IntArray(w * h)
        small.getPixels(px, 0, w, 0, 0, w, h)
        val tmp = IntArray(w * h)
        repeat(3) {
            boxPass(px, tmp, w, h, radius, horizontal = true)
            boxPass(tmp, px, w, h, radius, horizontal = false)
        }
        small.setPixels(px, 0, w, 0, 0, w, h)
        return small
    }

    /** One box-blur pass along rows or columns, edges clamped. */
    private fun boxPass(input: IntArray, out: IntArray, w: Int, h: Int, r: Int, horizontal: Boolean) {
        val lines = if (horizontal) h else w
        val len = if (horizontal) w else h
        val div = 2 * r + 1
        for (line in 0 until lines) {
            fun at(i: Int): Int {
                val c = i.coerceIn(0, len - 1)
                return if (horizontal) input[line * w + c] else input[c * w + line]
            }
            var sr = 0; var sg = 0; var sb = 0
            for (i in -r..r) {
                val c = at(i)
                sr += (c shr 16) and 0xFF; sg += (c shr 8) and 0xFF; sb += c and 0xFF
            }
            for (i in 0 until len) {
                val idx = if (horizontal) line * w + i else i * w + line
                out[idx] = (0xFF shl 24) or ((sr / div) shl 16) or ((sg / div) shl 8) or (sb / div)
                val add = at(i + r + 1)
                val sub = at(i - r)
                sr += ((add shr 16) and 0xFF) - ((sub shr 16) and 0xFF)
                sg += ((add shr 8) and 0xFF) - ((sub shr 8) and 0xFF)
                sb += (add and 0xFF) - (sub and 0xFF)
            }
        }
    }

    /** The [crop] part of [src] (0..1 fractions), scaled to [w] x [h]. */
    private fun region(src: Bitmap, crop: RectF, w: Int, h: Int): Bitmap {
        val l = (crop.left * src.width).roundToInt().coerceIn(0, src.width - 1)
        val t = (crop.top * src.height).roundToInt().coerceIn(0, src.height - 1)
        val r = (crop.right * src.width).roundToInt().coerceIn(l + 1, src.width)
        val b = (crop.bottom * src.height).roundToInt().coerceIn(t + 1, src.height)
        return Bitmap.createBitmap(src, l, t, r - l, b - t).scale(w, h)
    }

    private fun centerCrop(src: Bitmap, w: Int, h: Int): Bitmap {
        val s = max(w / src.width.toFloat(), h / src.height.toFloat())
        val sw = (src.width * s).roundToInt().coerceAtLeast(w)
        val sh = (src.height * s).roundToInt().coerceAtLeast(h)
        val scaled = if (sw == src.width && sh == src.height) src else src.scale(sw, sh)
        return Bitmap.createBitmap(scaled, (sw - w) / 2, (sh - h) / 2, w, h)
    }

    /** The full screen in pixels, system bars included (the wallpaper covers it all). */
    fun screenSize(context: Context): Pair<Int, Int> {
        val wm = context.getSystemService(WindowManager::class.java)
        return if (Build.VERSION.SDK_INT >= 30) {
            val b = wm.maximumWindowMetrics.bounds
            b.width() to b.height()
        } else {
            val m = android.util.DisplayMetrics()
            @Suppress("DEPRECATION")
            wm.defaultDisplay.getRealMetrics(m)
            m.widthPixels to m.heightPixels
        }
    }

    /** The home wallpaper's id, when Android lets this app read it (some versions do not). */
    private fun wallpaperId(context: Context): Int? = try {
        WallpaperManager.getInstance(context).getWallpaperId(WallpaperManager.FLAG_SYSTEM).takeIf { it > 0 }
    } catch (e: SecurityException) {
        null
    }

    // ------------------------------------------------------------------------------- drawing

    private val paint = Paint(Paint.FILTER_BITMAP_FLAG)
    private val dst = RectF()
    private val loc = IntArray(2)

    /**
     * Draws the blur behind [view] inside [clip] (view coordinates), lined up with the wallpaper
     * however the view is scrolled or scaled. Returns false when there is nothing to draw.
     */
    fun draw(canvas: Canvas, view: View, clip: Path, radius: Float = -1f): Boolean {
        val b = _active.value ?: return false
        view.getLocationOnScreen(loc)
        var scale = 1f
        var v: View? = view
        while (v != null) {
            scale *= v.scaleX
            v = v.parent as? View
        }
        if (scale <= 0f) return false
        // Glass on Android 13+: the lens shader ([radius] is the outline's corner radius).
        if (b.mode == RenderMode.GLASS && Build.VERSION.SDK_INT >= 33 &&
            GlassShader.draw(canvas, view, clip, radius, b.bitmap, b.screenW, b.screenH, loc, scale)
        ) {
            return true
        }
        canvas.withClip(clip) {
            scale(1f / scale, 1f / scale)
            translate(-loc[0].toFloat(), -loc[1].toFloat())
            dst.set(0f, 0f, b.screenW.toFloat(), b.screenH.toFloat())
            drawBitmap(b.bitmap, null, dst, paint)
        }
        return true
    }

    /**
     * How much of a surface's own colour to keep over the blur. A [dark] colour (dark mode)
     * keeps more, so folders and panels still read dark over a light wallpaper.
     */
    fun fillShare(dark: Boolean): Float = when (_active.value?.mode) {
        RenderMode.FROSTED -> if (dark) 0.85f else 0.55f
        RenderMode.GLASS -> if (dark) 0.65f else 0.3f
        else -> 1f
    }

    /** [fillShare] for an ARGB colour, judged by its brightness. */
    fun fillShareFor(color: Int): Float = fillShare(dark = androidx.core.graphics.ColorUtils.calculateLuminance(color or (0xFF shl 24)) < 0.5)

    /**
     * Redraws [view] whenever it moves on screen (page scroll, edit-mode zoom) while a blur is
     * shown, so the blur stays still under it. Install once, from the view's constructor.
     */
    fun track(view: View) {
        tracked.add(view)
        val last = IntArray(2) { Int.MIN_VALUE }
        val now = IntArray(2)
        var lastScale = 0f
        val listener = ViewTreeObserver.OnPreDrawListener {
            if (_active.value != null && !quiet()) {
                view.getLocationOnScreen(now)
                var s = 1f
                var v: View? = view
                while (v != null) { s *= v.scaleX; v = v.parent as? View }
                if (now[0] != last[0] || now[1] != last[1] || s != lastScale) {
                    last[0] = now[0]; last[1] = now[1]; lastScale = s
                    view.invalidate()
                }
            }
            true
        }
        view.addOnAttachStateChangeListener(object : View.OnAttachStateChangeListener {
            override fun onViewAttachedToWindow(v: View) = v.viewTreeObserver.addOnPreDrawListener(listener)
            override fun onViewDetachedFromWindow(v: View) = v.viewTreeObserver.removeOnPreDrawListener(listener)
        })
    }

    // Whole-screen animations (the Apps screen sliding over home, the unlock and return zoom,
    // edit mode's zoom) move every tracked view on every frame. Re-aligning each one's blur then
    // redrew every folder and widget, glass shader included, 60 times a second, and broke the
    // hardware layer those animations draw home through: smooth on the A54, choppy on the A03.
    // Meanwhile the blur moves with the screen (nobody sees it at that speed) and every view
    // re-aligns once at the end.
    private val tracked: MutableSet<View> = java.util.Collections.newSetFromMap(java.util.WeakHashMap())
    private var held = false
    private var quietUntil = 0L
    private val realign = Runnable { if (!quiet()) tracked.toList().forEach { it.invalidate() } }

    private fun quiet() = held || SystemClock.uptimeMillis() < quietUntil

    /** Holds re-alignment while [on] (a gesture-driven whole-screen move, such as the Apps screen). */
    fun hold(on: Boolean) {
        if (held == on) return
        held = on
        if (!on) main.post(realign)
    }

    /** No re-alignment for [ms] (a whole-screen animation of that length), then once. */
    fun quietFor(ms: Long) {
        quietUntil = maxOf(quietUntil, SystemClock.uptimeMillis() + ms)
        main.removeCallbacks(realign)
        main.postDelayed(realign, quietUntil - SystemClock.uptimeMillis() + 16)
    }
}
