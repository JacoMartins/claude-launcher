package io.github.jacomartins.canyonui.ui.design

import android.graphics.Canvas
import android.graphics.LinearGradient
import android.graphics.Paint
import android.graphics.RectF
import android.graphics.Shader
import kotlin.math.PI
import kotlin.math.cos
import kotlin.math.pow

/**
 * The interface's border ([OneDepthBorder]) for View-based surfaces (widgets, folders, edit mode
 * pages), drawn exactly like [oneDepthBorder]: a rim lit from the top, a weaker bounce light at
 * the bottom, and dark sides at the left and right, on every shape. Gradients are rebuilt only
 * when the size or mode changes.
 */
class DepthRingPainter(density: Float) {
    private val width = density
    private val ring = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = width
    }
    private val side = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = width
    }
    private val tmp = RectF()
    private var cachedW = -1f
    private var cachedH = -1f
    private var cachedR = -1f
    private var cachedTop = Float.NaN
    private var cachedLeft = Float.NaN
    private var cachedDark: Boolean? = null

    /** Draws the ring just inside [rect] with corner [radius] (clamped to a pill). */
    fun draw(canvas: Canvas, rect: RectF, radius: Float, dark: Boolean) {
        val w = rect.width()
        val h = rect.height()
        if (w <= 0f || h <= 0f) return
        val r = radius.coerceAtMost(minOf(w, h) / 2f)
        val s = width
        if (w != cachedW || h != cachedH || r != cachedR || dark != cachedDark || rect.top != cachedTop || rect.left != cachedLeft) {
            cachedW = w; cachedH = h; cachedR = r; cachedDark = dark; cachedTop = rect.top; cachedLeft = rect.left
            ring.shader = litGradient(rect.top, h, if (dark) BUTTON_ALPHA_DARK else BUTTON_ALPHA_LIGHT)
            side.shader = if (s > 0f) tipGradient(rect.left, w, r, if (dark) SIDE_ALPHA_DARK else SIDE_ALPHA_LIGHT) else null
        }
        val half = width / 2f
        tmp.set(rect.left + s + half, rect.top + half, rect.right - s - half, rect.bottom - half)
        val rr = (r - half).coerceAtLeast(0f)
        canvas.drawRoundRect(tmp, rr, rr, ring)
        if (s > 0f) {
            tmp.set(rect.left + s / 2f, rect.top + s / 2f, rect.right - s / 2f, rect.bottom - s / 2f)
            val sr = (r - s / 2f).coerceAtLeast(0f)
            canvas.drawRoundRect(tmp, sr, sr, side)
        }
    }

    /**
     * The same border along any outline (an icon-shaped folder): [bounds] is the outline's box,
     * for the gradients; [radius] is about the outline's corner radius, for the dark sides.
     */
    fun drawPath(canvas: Canvas, path: android.graphics.Path, bounds: RectF, dark: Boolean, radius: Float = bounds.width() * 0.3f) {
        val h = bounds.height()
        if (h <= 0f) return
        if (h != cachedH || bounds.top != cachedTop || bounds.left != cachedLeft || dark != cachedDark || cachedR != -2f) {
            cachedW = bounds.width(); cachedH = h; cachedR = -2f; cachedDark = dark; cachedTop = bounds.top; cachedLeft = bounds.left
            ring.shader = litGradient(bounds.top, h, if (dark) BUTTON_ALPHA_DARK else BUTTON_ALPHA_LIGHT)
            side.shader = tipGradient(bounds.left, bounds.width(), radius, if (dark) SIDE_ALPHA_DARK else SIDE_ALPHA_LIGHT)
        }
        canvas.drawPath(path, ring)
        canvas.drawPath(path, side)
    }

    private fun litGradient(top: Float, h: Float, alpha: Float): LinearGradient {
        val colors = IntArray(STEPS + 1)
        val stops = FloatArray(STEPS + 1)
        for (i in 0..STEPS) {
            val t = i / STEPS.toFloat()
            val a = maxOf(KEY * glow(t / SPREAD), FILL * glow((1f - t) / SPREAD)) * alpha
            colors[i] = ((a * 255).toInt().coerceIn(0, 255) shl 24) or 0xFFFFFF
            stops[i] = t
        }
        return LinearGradient(0f, top, 0f, top + h, colors, stops, Shader.TileMode.CLAMP)
    }

    private fun tipGradient(left: Float, w: Float, r: Float, alpha: Float): LinearGradient {
        val fade = (r / w).coerceAtMost(0.5f)
        val ease = floatArrayOf(1f, .84f, .5f, .16f, 0f)
        val pos = floatArrayOf(0f, .25f, .5f, .75f, 1f)
        val colors = ArrayList<Int>()
        val stops = ArrayList<Float>()
        for (i in pos.indices) { stops += fade * pos[i]; colors += black(ease[i] * alpha) }
        for (i in pos.indices.reversed()) { stops += 1f - fade * pos[i]; colors += black(ease[i] * alpha) }
        return LinearGradient(left, 0f, left + w, 0f, colors.toIntArray(), stops.toFloatArray(), Shader.TileMode.CLAMP)
    }

    private fun black(a: Float) = (a * 255).toInt().coerceIn(0, 255) shl 24

    private fun glow(x: Float): Float = if (x >= 1f) 0f else (0.5f + 0.5f * cos(PI.toFloat() * x)).pow(SOFTNESS)

    private companion object {
        const val STEPS = 32
        // The shared border's values (OneDepthBorder).
        val KEY = OneDepthBorder.Light.key
        val FILL = OneDepthBorder.Light.fill
        val SPREAD = OneDepthBorder.Light.spread
        val SOFTNESS = OneDepthBorder.Light.softness
        const val BUTTON_ALPHA_DARK = OneDepthBorder.ALPHA_DARK
        const val BUTTON_ALPHA_LIGHT = OneDepthBorder.ALPHA_LIGHT
        const val SIDE_ALPHA_DARK = OneDepthBorder.SIDE_ALPHA_DARK
        const val SIDE_ALPHA_LIGHT = OneDepthBorder.SIDE_ALPHA_LIGHT
    }
}
