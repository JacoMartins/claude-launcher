package io.github.jacomartins.canyonui.ui.design

import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import android.os.Build
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.runtime.staticCompositionLocalOf
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.unit.dp
import dev.chrisbanes.haze.HazeInput
import dev.chrisbanes.haze.HazeSourceSelection
import dev.chrisbanes.haze.HazeState
import dev.chrisbanes.haze.hazeSource
import dev.chrisbanes.haze.blur.HazeBlurStyle
import dev.chrisbanes.haze.blur.hazeBlur
import dev.chrisbanes.haze.glass.GlassStyle
import dev.chrisbanes.haze.glass.hazeGlass
import androidx.compose.ui.composed
import androidx.compose.ui.draw.drawBehind
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.luminance
import androidx.compose.ui.graphics.FilterQuality
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.Path
import androidx.compose.ui.graphics.addOutline
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.graphics.drawscope.clipPath
import androidx.compose.ui.layout.onGloballyPositioned
import androidx.compose.ui.layout.positionInRoot
import androidx.compose.ui.platform.LocalView
import androidx.compose.ui.unit.IntOffset
import androidx.compose.ui.unit.IntSize
import io.github.jacomartins.canyonui.data.RenderMode

/** The Haze state the overlay's wallpaper copy feeds; null where there is no copy to sample. */
val LocalBackdropHaze = staticCompositionLocalOf<HazeState?> { null }

/**
 * The wallpaper copy as Haze's source, full screen and lined up with the real wallpaper. It is
 * recorded for the effects but never shown (the real wallpaper and home are what is seen), so it
 * sits under an invisible layer. Put it first in the overlay. [content], when given, is a
 * screen-sized picture used instead (the home screen behind an open folder), stretched over it.
 */
@Composable
fun BackdropSource(state: HazeState, content: ImageBitmap? = null, offsetY: () -> Float = { 0f }) {
    val src by WallpaperBackdrop.source.collectAsState()
    val b = src ?: return
    val image = remember(b) { b.asImageBitmap() }
    if (content != null) {
        Box(Modifier.fillMaxSize().graphicsLayer { alpha = 0f }) {
            Canvas(Modifier.fillMaxSize().hazeSource(state, zIndex = 0f, key = WALLPAPER_SOURCE)) {
                drawImage(content, dstSize = IntSize(size.width.toInt(), size.height.toInt()), filterQuality = FilterQuality.Low)
            }
        }
        return
    }
    // Drawn at the view's own origin, less [offsetY]: the home overlay fills the screen from its
    // top-left, and the Apps screen passes how far it is slid down (read while drawing, so the
    // copy moves the other way and the glass slides over a still wallpaper).
    Box(Modifier.fillMaxSize().graphicsLayer { alpha = 0f }) {
        Canvas(Modifier.fillMaxSize().hazeSource(state, zIndex = 0f, key = WALLPAPER_SOURCE)) {
            drawImage(image, dstOffset = IntOffset(0, -offsetY().toInt()), dstSize = IntSize(b.width, b.height))
        }
    }
}

/**
 * Frosted and glass modes: the wallpaper behind this element, inside [shape]. In the home
 * overlay (with a [BackdropSource]) Haze draws it, as Haze's glass in the glass mode and as
 * Haze's blur in the frosted mode; elsewhere a pre-blurred copy lined up with the wallpaper is
 * drawn instead. Does nothing in the translucent mode. Put it before the element's own
 * (see-through) background.
 */
fun Modifier.backdrop(shape: RoundedCornerShape, withContent: Boolean = false): Modifier = composed {
    val blur by WallpaperBackdrop.active.collectAsState()
    val b = blur ?: return@composed this
    val haze = LocalBackdropHaze.current
    if (haze != null && Build.VERSION.SDK_INT >= 31) {
        // The wallpaper alone, or ([withContent]) with the content marked as a source too (the
        // Apps screen's icons under its search field).
        // The frosted blur over the apps came out dark (their source is see-through), so only
        // glass takes the content.
        val input = if (withContent && b.mode == RenderMode.GLASS) {
            HazeInput.Sources(haze, HazeSourceSelection.All)
        } else {
            HazeInput.Sources(haze, HazeSourceSelection.All.where { it.key == WALLPAPER_SOURCE })
        }
        return@composed if (b.mode == RenderMode.GLASS) {
            // No rim light or edge shadow of Haze's own: the border is the interface's depth
            // ring, the same in every mode. More bending at the rim and less blur than Haze's
            // defaults (which are 0.7, a quarter of the side, 15dp and 14dp), like the views'
            // glass lens ([GlassShader]).
            val style = remember(shape) {
                GlassStyle.regular.then {
                    shape(shape)
                    specularIntensity(0f)
                    edgeShadow(Color.Transparent)
                    optics(
                        refractionStrength = 1f,
                        refractionHeightFraction = LiquidGlass.BAND_SHARE,
                        refractionDisplacement = LiquidGlass.BEND_DP.dp,
                        blurRadius = 6.dp,
                    )
                }
            }
            this.hazeGlass(input, style = style)
        } else {
            this.clip(shape).hazeBlur(input, style = FrostedStyle)
        }
    }
    val view = LocalView.current
    val image = remember(b) { b.bitmap.asImageBitmap() }
    var origin by remember { mutableStateOf(Offset.Zero) }
    val path = remember { Path() }
    this
        .onGloballyPositioned { c ->
            val loc = IntArray(2)
            view.getLocationOnScreen(loc)
            origin = c.positionInRoot() + Offset(loc[0].toFloat(), loc[1].toFloat())
        }
        .drawBehind {
            path.reset()
            path.addOutline(shape.createOutline(size, layoutDirection, this))
            clipPath(path) {
                drawImage(
                    image,
                    dstOffset = IntOffset(-origin.x.toInt(), -origin.y.toInt()),
                    dstSize = IntSize(b.screenW, b.screenH),
                    filterQuality = FilterQuality.Low,
                )
            }
        }
}

/** Haze source key of the wallpaper copy. */
const val WALLPAPER_SOURCE = "wallpaper"

private val FrostedStyle = HazeBlurStyle {
    blurRadius(28.dp)
    noiseFactor(0.1f)
}

/** [color] with less of its opacity when a blur shows through (frosted keeps more than glass). */
@Composable
fun backdropColor(color: Color): Color {
    val blur by WallpaperBackdrop.active.collectAsState()
    return if (blur == null) color else color.copy(alpha = color.alpha * WallpaperBackdrop.fillShare(dark = color.luminance() < 0.5f))
}
