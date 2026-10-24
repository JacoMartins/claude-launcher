package io.github.jacomartins.canyonui.ui.home

import android.graphics.Canvas
import android.graphics.Paint
import android.graphics.Rect
import android.graphics.RectF
import android.graphics.Typeface

/**
 * A notification badge at an icon's top-right corner: a dot, or a pill with the count (up to
 * 999, then "999+"). Sized from the icon so it scales with the grid.
 */
class BadgePainter {
    private val fill = Paint(Paint.ANTI_ALIAS_FLAG).apply { color = BADGE_COLOR }
    private val text = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = 0xFFFFFFFF.toInt()
        textAlign = Paint.Align.CENTER
        typeface = Typeface.create(Typeface.DEFAULT, 700, false)
    }
    private val pill = RectF()

    /** [count] above 0 draws a badge on [icon]; [numbers] false draws only a dot. */
    fun draw(canvas: Canvas, icon: Rect, count: Int, numbers: Boolean) {
        if (count <= 0 || icon.isEmpty) return
        val size = icon.width().toFloat()
        if (!numbers) {
            val r = size * DOT_SHARE / 2f
            canvas.drawCircle(icon.right - r * 0.9f, icon.top + r * 0.9f, r, fill)
            return
        }
        val h = size * NUMBER_SHARE
        text.textSize = h * 0.62f
        val label = if (count > 999) "999+" else count.toString()
        val w = maxOf(h, text.measureText(label) + h * 0.5f)
        // Its top-right sits just outside the icon's corner, like the system's badges.
        val right = icon.right + h * 0.18f
        val top = icon.top - h * 0.18f
        pill.set(right - w, top, right, top + h)
        canvas.drawRoundRect(pill, h / 2f, h / 2f, fill)
        val baseline = pill.centerY() - (text.descent() + text.ascent()) / 2f
        canvas.drawText(label, pill.centerX(), baseline, text)
    }

    private companion object {
        const val BADGE_COLOR = 0xFFF2542D.toInt()
        const val NUMBER_SHARE = 0.36f
        const val DOT_SHARE = 0.22f
    }
}
