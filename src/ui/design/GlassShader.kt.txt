package io.github.jacomartins.canyonui.ui.design

import android.graphics.Bitmap
import android.graphics.BitmapShader
import android.graphics.Canvas
import android.graphics.Matrix
import android.graphics.Paint
import android.graphics.Path
import android.graphics.RectF
import android.graphics.RuntimeShader
import android.graphics.Shader
import android.view.View
import androidx.annotation.RequiresApi
import java.util.WeakHashMap

/**
 * Liquid glass for the View-based surfaces (folders, edit pages, widget cards), where Haze (a
 * Compose library) cannot draw: an AGSL lens over the lightly blurred wallpaper. Near the rim
 * the glass bends what is behind it inward (the edge magnifies, like a thick pane). No light of
 * its own: the border is the same depth ring as everywhere else. Android 13+.
 */
/**
 * The glass's optics, shared by the views' lens ([GlassShader]) and the panels' Haze glass.
 * Stronger than Haze's default (15dp over a quarter of the shorter side): a thinner rim that
 * bends more.
 */
object LiquidGlass {
    const val BEND_DP = 32f
    const val BAND_SHARE = 0.2f
}

@RequiresApi(33)
internal object GlassShader {

    private const val SRC = """
        uniform shader image;
        uniform float2 origin;
        uniform float2 size;
        uniform float radius;
        uniform float bend;
        uniform float band;

        float sdBox(float2 p, float2 b, float r) {
            float2 q = abs(p) - b + float2(r);
            return length(max(q, float2(0.0))) + min(max(q.x, q.y), 0.0) - r;
        }

        half4 main(float2 xy) {
            float2 hs = size * 0.5;
            float2 p = xy - origin - hs;
            float r = min(radius, min(hs.x, hs.y));
            float d = sdBox(p, hs, r);
            // How deep into the rim band: 1 at the edge, 0 where the flat middle starts. The
            // band is a share of the shorter side, so every shape bends the same way all round.
            float bw = min(band, min(hs.x, hs.y));
            float t = clamp(1.0 + d / bw, 0.0, 1.0);
            float2 e = float2(0.75, 0.0);
            float2 g = float2(sdBox(p + e.xy, hs, r) - sdBox(p - e.xy, hs, r),
                              sdBox(p + e.yx, hs, r) - sdBox(p - e.yx, hs, r));
            float2 n = g / max(length(g), 0.0001);
            // A rounded (circular) lens profile: flat in the middle, steepest at the rim.
            float lens = 1.0 - sqrt(max(1.0 - t * t, 0.0));
            float2 shift = n * (lens * bend);
            // A touch of colour separation at the rim, as light splits in thick glass.
            half3 c = half3(
                image.eval(xy - shift * 1.06).r,
                image.eval(xy - shift).g,
                image.eval(xy - shift * 0.94).b);
            return half4(c, 1.0);
        }
    """

    /** One shader and paint per view: a recorded draw keeps the values it was drawn with. */
    private class Holder(val shader: RuntimeShader, val paint: Paint) {
        var image: Bitmap? = null
        var imageShader: BitmapShader? = null
    }

    private val holders = WeakHashMap<View, Holder>()
    private val matrix = Matrix()
    private val bounds = RectF()

    /**
     * Draws the glass inside [clip] on [view]'s canvas. [bitmap] is the blurred wallpaper (drawn
     * stretched to [screenW] x [screenH]); [loc] and [scale] place the view on screen.
     */
    fun draw(
        canvas: Canvas,
        view: View,
        clip: Path,
        radius: Float,
        bitmap: Bitmap,
        screenW: Int,
        screenH: Int,
        loc: IntArray,
        scale: Float,
    ): Boolean {
        // Runtime shaders draw only on the GPU; snapshots (drag images) are drawn in software.
        if (broken || !canvas.isHardwareAccelerated) return false
        val h = holders[view] ?: try {
            Holder(RuntimeShader(SRC), Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG)).also { holders[view] = it }
        } catch (e: IllegalArgumentException) {
            // The GPU's shader compiler refused it: fall back to the plain blur everywhere.
            android.util.Log.w("GlassShader", "Glass shader unavailable", e)
            broken = true
            return false
        }
        if (h.image !== bitmap) {
            h.image = bitmap
            // Smooth (bilinear) sampling: the small blurred copy is stretched to the screen and
            // magnified at the rim, and nearest-pixel sampling showed it as blocks.
            h.imageShader = BitmapShader(bitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP).apply {
                filterMode = BitmapShader.FILTER_MODE_LINEAR
            }
        }
        val image = h.imageShader ?: return false
        // Bitmap pixels -> screen -> this view's coordinates.
        matrix.reset()
        matrix.postScale(screenW / bitmap.width.toFloat(), screenH / bitmap.height.toFloat())
        matrix.postTranslate(-loc[0].toFloat(), -loc[1].toFloat())
        matrix.postScale(1f / scale, 1f / scale)
        image.setLocalMatrix(matrix)
        clip.computeBounds(bounds, true)
        val d = view.resources.displayMetrics.density
        val s = h.shader
        s.setInputShader("image", image)
        s.setFloatUniform("origin", bounds.left, bounds.top)
        s.setFloatUniform("size", bounds.width(), bounds.height())
        s.setFloatUniform("radius", if (radius >= 0f) radius else minOf(bounds.width(), bounds.height()) * 0.25f)
        s.setFloatUniform("bend", LiquidGlass.BEND_DP * d)
        s.setFloatUniform("band", minOf(bounds.width(), bounds.height()) * LiquidGlass.BAND_SHARE)
        h.paint.shader = s
        canvas.drawPath(clip, h.paint)
        return true
    }

    private var broken = false


}
