package io.github.jacomartins.canyonui.ui.home

import android.content.Context
import android.graphics.Canvas
import android.graphics.Paint
import android.graphics.Path
import android.view.View
import io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop
import io.github.jacomartins.canyonui.util.Motion

/**
 * Edit mode's background, behind the pages: the Transparency effect over the whole screen (the
 * blurred wallpaper in frosted and glass, under a tint by dark or light mode). The pages
 * themselves stay clear. Fades in when edit mode opens and out when it closes.
 */
class EditBackdropView(context: Context) : View(context) {
    private val tint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val path = Path()

    /** The tint over the effect (ARGB), by dark or light mode. */
    var tintColor = 0x47000000
        set(value) { if (field != value) { field = value; invalidate() } }

    init {
        alpha = 0f
        visibility = GONE
        importantForAccessibility = IMPORTANT_FOR_ACCESSIBILITY_NO
        WallpaperBackdrop.track(this)
    }

    /** Fades the background in ([show]) or out. */
    fun show(show: Boolean) {
        animate().cancel()
        if (show) visibility = VISIBLE
        val end = { if (!show) visibility = GONE }
        if (Motion.enabled()) {
            animate().alpha(if (show) 1f else 0f).setDuration(FADE_MS).withLayer().withEndAction(end).start()
        } else {
            alpha = if (show) 1f else 0f
            end()
        }
    }

    override fun onDraw(canvas: Canvas) {
        path.reset()
        path.addRect(0f, 0f, width.toFloat(), height.toFloat(), Path.Direction.CW)
        tint.color = tintColor
        if (WallpaperBackdrop.draw(canvas, this, path, 0f)) {
            tint.alpha = (tint.alpha * WallpaperBackdrop.fillShareFor(tintColor)).toInt()
        }
        canvas.drawRect(0f, 0f, width.toFloat(), height.toFloat(), tint)
    }

    private companion object {
        const val FADE_MS = 240L
    }
}
