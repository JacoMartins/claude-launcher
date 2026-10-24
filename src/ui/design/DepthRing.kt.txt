package io.github.jacomartins.canyonui.ui.design

import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.ReadOnlyComposable
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.drawWithCache
import androidx.compose.ui.geometry.CornerRadius
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.Brush
import androidx.compose.ui.graphics.BlendMode
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.Paint
import androidx.compose.ui.graphics.Path
import androidx.compose.ui.geometry.Rect
import androidx.compose.ui.geometry.RoundRect
import androidx.compose.ui.graphics.drawscope.Stroke
import androidx.compose.ui.graphics.luminance
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp
import kotlin.math.PI
import kotlin.math.cos
import kotlin.math.pow

/*
 * The depth ring from the OneDepth design system (its "angular border"): a rim lit from the top,
 * a weaker bounce light at the bottom, and dark crescents at the left and right tips. Used on
 * interface surfaces only (buttons, search fields, bars, cards), never on app icons.
 *
 * Differences from the OneDepth original: the ring is plain translucent white (light) and black
 * (tips) drawn straight over the surface, with no blend modes and no offscreen mask layer, so it
 * cannot glitch where layers meet; brushes are built once per size in drawWithCache (no per-frame
 * allocation); rounded rectangles are supported as well as pills.
 */

private const val STEPS = 32

/** Vertical light: [key] hits the top edge, [fill] bounces off the bottom. Scales with height. */
data class DepthLight(
    val key: Float = 0.9f,
    val fill: Float = 0.35f,
    val spread: Float = 0.6f,
    val softness: Float = 0.65f,
) {
    companion object {
        /** Wide, long surfaces such as bars and search fields. */
        val Surface = DepthLight(key = 0.9f, fill = 0.35f, spread = 0.6f)

        /** Short surfaces such as buttons: a tighter glow. */
        val Button = DepthLight(key = 0.8f, fill = 0.25f, spread = 0.5f)
    }
}

// 1 at x = 0, easing to 0 at x >= 1.
private fun glow(x: Float, softness: Float): Float =
    if (x >= 1f) 0f else (0.5f + 0.5f * cos(PI.toFloat() * x)).pow(softness)

private fun DepthLight.brush(height: Float): Brush {
    val s = spread.coerceAtLeast(0.01f)
    return Brush.verticalGradient(
        colorStops = Array(STEPS + 1) { i ->
            val t = i / STEPS.toFloat()
            val a = maxOf(key * glow(t / s, softness), fill * glow((1f - t) / s, softness))
            t to Color.White.copy(alpha = a)
        },
        startY = 0f,
        endY = height,
    )
}

private val TIP_EASE = listOf(0f to 1f, .25f to .84f, .5f to .5f, .75f to .16f, 1f to 0f)

/**
 * The one border of the interface (OneDepth's button border): every surface with a border uses
 * these values, in Compose ([oneDepthBorder]) and in Views ([DepthRingPainter]), in every mode.
 */
object OneDepthBorder {
    val Width = 1.dp
    val Light = DepthLight.Button
    const val ALPHA_DARK = 0.40f
    const val ALPHA_LIGHT = 0.60f
    const val SIDE_ALPHA_DARK = 0.40f
    const val SIDE_ALPHA_LIGHT = 0.25f
}

/**
 * Draws the depth ring over the content's bounds.
 * [cornerRadius] null means a pill (radius = half the height).
 */
private fun Modifier.depthRing(
    cornerRadius: Dp? = null,
    width: Dp = 1.dp,
    alpha: Float = 0.333f,
    light: DepthLight = DepthLight.Surface,
    sideWidth: Dp = 1.dp,
    sideAlpha: Float = 0.20f,
    sideColor: Color = Color.Black,
): Modifier = this
    .drawWithCache {
        val w = width.toPx()
        val sidePx = sideWidth.toPx()
        val h = size.height
        val r = (cornerRadius?.toPx() ?: (h / 2f)).coerceAtMost(minOf(h, size.width) / 2f)
        // The dark sides on every shape, square and round ones included.
        val s = sidePx

        val ringBrush = light.brush(h)
        val ringTopLeft = Offset(s + w / 2f, w / 2f)
        val ringSize = Size((size.width - 2f * s - w).coerceAtLeast(0f), h - w)
        val ringRadius = CornerRadius((r - w / 2f).coerceAtLeast(0f))
        val ringStroke = Stroke(w)

        val sideBrush = if (s > 0.01f && size.width > 0f) {
            val fade = (r / size.width).coerceAtMost(0.5f)
            val stops = TIP_EASE.map { (t, a) -> fade * t to sideColor.copy(alpha = a) } +
                TIP_EASE.reversed().map { (t, a) -> 1f - fade * t to sideColor.copy(alpha = a) }
            Brush.horizontalGradient(colorStops = stops.toTypedArray(), startX = 0f, endX = size.width)
        } else {
            null
        }
        val sideTopLeft = Offset(s / 2f, s / 2f)
        val sideSize = Size(size.width - s, h - s)
        val sideRadius = CornerRadius((r - s / 2f).coerceAtLeast(0f))
        val sideStroke = Stroke(s)

        onDrawWithContent {
            drawContent()
            // 1) Lit ring, inset by the side stroke so a crescent opens at each tip.
            drawRoundRect(ringBrush, ringTopLeft, ringSize, ringRadius, style = ringStroke, alpha = alpha)
            // 2) Dark tips easing to transparent along the straight edges.
            if (sideBrush != null) {
                drawRoundRect(sideBrush, sideTopLeft, sideSize, sideRadius, style = sideStroke, alpha = sideAlpha)
            }
        }
    }

/** True when the current theme's surfaces are dark. */
@Composable
@ReadOnlyComposable
fun isDarkSurface(): Boolean = MaterialTheme.colorScheme.surface.luminance() < 0.5f

/**
 * The interface's border ([OneDepthBorder]) for any Compose surface: buttons, fields, bars,
 * panels, cards. [cornerRadius] null means a pill.
 */
@Composable
fun Modifier.oneDepthBorder(cornerRadius: Dp? = null): Modifier {
    val dark = isDarkSurface()
    return depthRing(
        cornerRadius = cornerRadius,
        width = OneDepthBorder.Width,
        alpha = if (dark) OneDepthBorder.ALPHA_DARK else OneDepthBorder.ALPHA_LIGHT,
        light = OneDepthBorder.Light,
        sideWidth = OneDepthBorder.Width,
        sideAlpha = if (dark) OneDepthBorder.SIDE_ALPHA_DARK else OneDepthBorder.SIDE_ALPHA_LIGHT,
    )
}

/**
 * [OneDepthBorder] along the top edge and top corners only, for a sheet that runs off the bottom
 * of the screen (the Apps screen): the lit rim and the dark corner tips, fading out down the sides.
 * [radius] gives the live corner radius in px for the surface height, read at draw time; the border fades with it, so a
 * sheet that has gone square shows none.
 */
@Composable
fun Modifier.oneDepthTopBorder(maxRadius: Dp, radius: (height: Float) -> Float): Modifier {
    val dark = isDarkSurface()
    val alpha = if (dark) OneDepthBorder.ALPHA_DARK else OneDepthBorder.ALPHA_LIGHT
    val sideAlpha = if (dark) OneDepthBorder.SIDE_ALPHA_DARK else OneDepthBorder.SIDE_ALPHA_LIGHT
    val light = OneDepthBorder.Light
    return drawWithCache {
        val w = OneDepthBorder.Width.toPx()
        val maxR = maxRadius.toPx()
        // How far down the sides the border reaches before it is gone.
        val fadeEnd = (maxR * 3f).coerceAtMost(size.height)
        val ringBrush = Brush.verticalGradient(
            colorStops = Array(STEPS + 1) { i ->
                val t = i / STEPS.toFloat()
                t to Color.White.copy(alpha = light.key * glow(t / light.spread, light.softness))
            },
            startY = 0f,
            endY = fadeEnd,
        )
        val mask = Brush.verticalGradient(0f to Color.Black, (maxR / fadeEnd) to Color.Black, 1f to Color.Transparent, startY = 0f, endY = fadeEnd)
        val layer = Paint()
        val path = Path()
        val stroke = Stroke(w)
        val layerBounds = Rect(0f, 0f, size.width, fadeEnd)
        // The tips' gradient depends on the radius: kept while it stays the same (the sheet
        // resting, or a radius already seen), not rebuilt on every frame of a slide.
        var tipsFor = -1f
        var tips: Brush? = null
        onDrawWithContent {
            drawContent()
            val r = radius(size.height).coerceIn(0f, maxR)
            val share = if (maxR > 0f) r / maxR else 0f
            if (share < 0.01f) return@onDrawWithContent
            if (r != tipsFor || tips == null) {
                tipsFor = r
                val fade = (r / size.width).coerceAtMost(0.5f)
                val tipStops = TIP_EASE.map { (t, a) -> fade * t to Color.Black.copy(alpha = a) } +
                    TIP_EASE.reversed().map { (t, a) -> 1f - fade * t to Color.Black.copy(alpha = a) }
                tips = Brush.horizontalGradient(colorStops = tipStops.toTypedArray(), startX = 0f, endX = size.width)
            }
            fun outline(inset: Float, left: Float, right: Float) {
                val cr = CornerRadius((r - inset).coerceAtLeast(0f))
                path.rewind()
                path.addRoundRect(
                    RoundRect(Rect(left, inset, right, size.height + r + w), topLeft = cr, topRight = cr),
                )
            }
            drawContext.canvas.saveLayer(layerBounds, layer)
            // Lit rim, inset by the tip stroke so a crescent opens at each corner (as in depthRing).
            outline(w / 2f, w + w / 2f, size.width - w - w / 2f)
            drawPath(path, ringBrush, alpha = alpha * share, style = stroke)
            // Dark tips at the corners easing to transparent along the top edge.
            outline(w / 2f, w / 2f, size.width - w / 2f)
            drawPath(path, tips!!, alpha = sideAlpha * share, style = stroke)
            // Fade both down the sides.
            drawRect(mask, size = Size(size.width, fadeEnd), blendMode = BlendMode.DstIn)
            drawContext.canvas.restore()
        }
    }
}
