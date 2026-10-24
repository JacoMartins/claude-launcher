package io.github.jacomartins.canyonui.data

import android.graphics.Path
import android.graphics.RectF
import kotlin.math.abs
import kotlin.math.cos
import kotlin.math.pow
import kotlin.math.sign
import kotlin.math.sin

/**
 * The outline given to app icons. [SYSTEM] keeps the icons exactly as the phone hands them out
 * (its own shape, and whatever its theme adds); every other shape is built from the app's own
 * icon layers, so it also leaves out effects a device bakes in.
 */
enum class IconShape(val storageKey: String) {
    SYSTEM("system"),
    CIRCLE("circle"),
    SQUIRCLE("squircle"),
    ROUNDED("rounded"),
    SQUARE("square"),
    ;

    /** The outline in a [size] x [size] square, or null for [SYSTEM]. */
    fun path(size: Float, out: Path = Path()): Path? {
        out.reset()
        when (this) {
            SYSTEM -> return null
            CIRCLE -> out.addOval(RectF(0f, 0f, size, size), Path.Direction.CW)
            ROUNDED -> out.addRoundRect(RectF(0f, 0f, size, size), size * 0.3f, size * 0.3f, Path.Direction.CW)
            SQUARE -> out.addRoundRect(RectF(0f, 0f, size, size), size * 0.08f, size * 0.08f, Path.Direction.CW)
            SQUIRCLE -> {
                // Superellipse |x|^n + |y|^n = 1 with n = 4: the "squircle".
                val r = size / 2f
                val steps = 96
                for (i in 0..steps) {
                    val t = (i.toDouble() / steps) * 2 * Math.PI
                    val c = cos(t)
                    val s = sin(t)
                    val x = r + r * (sign(c) * abs(c).pow(0.5)).toFloat()
                    val y = r + r * (sign(s) * abs(s).pow(0.5)).toFloat()
                    if (i == 0) out.moveTo(x, y) else out.lineTo(x, y)
                }
                out.close()
            }
        }
        return out
    }

    companion object {
        fun fromKey(key: String?) = entries.firstOrNull { it.storageKey == key } ?: SYSTEM
    }
}

/**
 * How icons are drawn, shared by both layout modes.
 *
 * @param themed Android 13 themed icons: apps that ship a monochrome layer are drawn in the
 *   wallpaper's colours.
 * @param packStyle When an icon pack defines a back plate, mask, overlay and scale (its
 *   iconback / iconmask / iconupon / scale entries), use them for apps the pack does not cover.
 */
data class IconStyle(
    val shape: IconShape = IconShape.SYSTEM,
    val themed: Boolean = false,
    val packStyle: Boolean = true,
)

/**
 * A user's own choices for one app: a label, an icon (from a pack, or an image file), and the
 * category it is organized under (an [io.github.jacomartins.canyonui.model.AppCategories.Category] key).
 */
data class IconOverride(
    val label: String? = null,
    val packPackage: String? = null,
    val drawable: String? = null,
    val file: String? = null,
    val category: String? = null,
) {
    val hasIcon get() = file != null || (packPackage != null && drawable != null)
    val isEmpty get() = label.isNullOrBlank() && !hasIcon && category == null
}
