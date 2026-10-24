package io.github.jacomartins.canyonui.ui.widget

import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.ColorFilter
import android.graphics.Outline
import android.graphics.Paint
import android.graphics.Path
import android.graphics.PixelFormat
import android.graphics.PorterDuff
import android.graphics.PorterDuffXfermode
import android.graphics.Rect
import android.graphics.RectF
import android.graphics.drawable.Drawable
import android.view.View
import android.view.ViewGroup
import android.view.ViewOutlineProvider
import android.content.res.Configuration
import io.github.jacomartins.canyonui.ui.design.DepthRingPainter
import androidx.core.graphics.createBitmap
import kotlin.math.min

/**
 * The design system's shape for every widget on home: corners rounded to [MAX_RADIUS_DP], or a
 * full pill for widgets one cell wide or one row tall, whatever shape the widget draws itself;
 * and, optionally, OneDepth's translucent ring along the edge. Applied to the top-level view (a
 * widget, a stack, a placeholder); widgets inside a stack are left plain.
 *
 * Many widgets draw their visible card smaller than the area they are given (a search bar with
 * transparent margins). The frame follows that card when it can be found: the first view down
 * a single-child chain that has a background.
 */
object WidgetFrame {
    const val MAX_RADIUS_DP = 26f

    /** Gap kept around a widget inside its cells, per side (measured on One UI 8.5: about 8.5dp). */
    const val INSET_DP = 8.5f

    /** Pills (one cell wide or one row tall) are at most this thick (One UI 8.5: about 76dp). */
    const val PILL_MAX_DP = 74f

    /** [pill]: a fully rounded end (one cell wide or one row tall). */
    fun apply(view: View, ring: Boolean, pill: Boolean) {
        val d = view.resources.displayMetrics.density
        val frame = view.outlineProvider as? Frame ?: Frame(MAX_RADIUS_DP * d).also { view.outlineProvider = it }
        view.clipToOutline = true
        if (frame.pill != pill) {
            frame.pill = pill
            view.invalidateOutline()
            view.foreground?.invalidateSelf()
        }
        val fg = view.foreground
        if (ring && fg !is Ring) {
            view.foreground = Ring(frame, d) {
                view.resources.configuration.uiMode and Configuration.UI_MODE_NIGHT_MASK == Configuration.UI_MODE_NIGHT_YES
            }
        }
        if (!ring && fg is Ring) view.foreground = null
        refresh(view)
    }

    fun clear(view: View) {
        if (view.outlineProvider is Frame) view.outlineProvider = ViewOutlineProvider.BACKGROUND
        view.clipToOutline = false
        if (view.foreground is Ring) view.foreground = null
    }

    /**
     * Re-finds the widget's visible card after a layout pass or a new widget layout. The pixel
     * check runs after the pass (never inside it), at most once per frame.
     */
    fun refresh(view: View) {
        val frame = view.outlineProvider as? Frame ?: return
        val old = Rect(frame.content)
        // The last pixel check stays valid while the widget keeps its size.
        val measured = frame.measured
        if (measured != null && frame.measuredW == view.width && frame.measuredH == view.height) {
            frame.content.set(measured)
        } else {
            contentBounds(view, frame.content)
        }
        if (old != frame.content) {
            view.invalidateOutline()
            view.foreground?.invalidateSelf()
        }
        if (view is LauncherWidgetHostView && !frame.measurePending) {
            frame.measurePending = true
            view.post {
                frame.measurePending = false
                val card = drawnCard(view) ?: return@post
                frame.measured = card
                frame.measuredW = view.width
                frame.measuredH = view.height
                if (card != frame.content) {
                    frame.content.set(card)
                    view.invalidateOutline()
                    view.foreground?.invalidateSelf()
                }
            }
        }
    }

    /**
     * The bounds of what the widget actually paints (alpha above a threshold), from a
     * quarter-size snapshot. Only a solid card counts: it must cover [CARD_MIN_SHARE] of the
     * area; widgets that are just text or icons on nothing keep the whole area.
     */
    private fun drawnCard(view: View): Rect? {
        val w = view.width
        val h = view.height
        if (w < 8 || h < 8) return null
        val sw = (w / SAMPLE).coerceAtLeast(1)
        val sh = (h / SAMPLE).coerceAtLeast(1)
        val bmp = createBitmap(sw, sh)
        val fg = view.foreground
        try {
            view.foreground = null // The ring itself must not count.
            snapshotting = true // Nor the blur drawn behind the card.
            val c = Canvas(bmp)
            c.scale(sw.toFloat() / w, sh.toFloat() / h)
            view.draw(c)
        } catch (e: RuntimeException) {
            // Hardware bitmaps cannot be drawn in software; keep the structural guess.
            return null
        } finally {
            view.foreground = fg
            snapshotting = false
        }
        val px = IntArray(sw * sh)
        bmp.getPixels(px, 0, sw, 0, 0, sw, sh)
        bmp.recycle()
        var l = sw
        var t = sh
        var r = -1
        var b = -1
        for (y in 0 until sh) for (x in 0 until sw) {
            if ((px[y * sw + x] ushr 24) > ALPHA_MIN) {
                if (x < l) l = x
                if (x > r) r = x
                if (y < t) t = y
                if (y > b) b = y
            }
        }
        if (r < 0) return null
        val area = (r - l + 1) * (b - t + 1)
        if (area < sw * sh * CARD_MIN_SHARE) return null
        return Rect(l * w / sw, t * h / sh, (r + 1) * w / sw, (b + 1) * h / sh)
    }

    /**
     * The widget's own card (measured from what it draws) as a rounded outline in [out], and
     * its corner radius; -1 when the widget draws no solid card (text or icons on nothing).
     */
    fun cardPath(view: View, out: Path): Float {
        if (snapshotting) return -1f
        val frame = view.outlineProvider as? Frame ?: return -1f
        val card = frame.measured ?: return -1f
        if (frame.measuredW != view.width || frame.measuredH != view.height) return -1f
        val r = frame.radius()
        out.reset()
        out.addRoundRect(RectF(card), r, r, Path.Direction.CW)
        return r
    }

    /**
     * The frame's outline (its card, or the whole view) in [out], and its corner radius; -1
     * when [view] has no frame. For stacks, whose widgets inside are plain.
     */
    fun framePath(view: View, out: Path): Float {
        if (snapshotting) return -1f
        val frame = view.outlineProvider as? Frame ?: return -1f
        val c = if (frame.content.isEmpty) Rect(0, 0, view.width, view.height) else frame.content
        val r = frame.radius()
        out.reset()
        out.addRoundRect(RectF(c), r, r, Path.Direction.CW)
        return r
    }

    private var snapshotting = false

    private const val SAMPLE = 4
    private const val ALPHA_MIN = 40
    private const val CARD_MIN_SHARE = 0.4f

    /**
     * A drag image of a framed widget: drawing a view into a bitmap skips its outline clip, so the
     * same rounded shape is cut out here. [local] is the part of the view in the bitmap.
     */
    fun clipSnapshot(view: View, bitmap: Bitmap, local: RectF) {
        val frame = view.outlineProvider as? Frame ?: return
        val c = Canvas(bitmap)
        c.translate(-local.left, -local.top)
        val r = frame.radius()
        val path = Path().apply {
            addRect(RectF(local), Path.Direction.CW)
            addRoundRect(RectF(frame.content), r, r, Path.Direction.CCW)
        }
        c.drawPath(path, Paint(Paint.ANTI_ALIAS_FLAG).apply { xfermode = PorterDuffXfermode(PorterDuff.Mode.CLEAR) })
    }

    private fun contentBounds(view: View, out: Rect) {
        out.set(0, 0, view.width, view.height)
        if (view !is LauncherWidgetHostView) return
        var cur: View = view
        var dx = 0
        var dy = 0
        while (cur is ViewGroup && cur.childCount == 1) {
            val c = cur.getChildAt(0)
            dx += c.left
            dy += c.top
            if (c.background != null) {
                val w = c.width
                val h = c.height
                // Only a substantial card counts; a small badge with a background must not shrink the frame.
                if (w * h >= view.width * view.height * 2 / 5) out.set(dx, dy, dx + w, dy + h)
                return
            }
            cur = c
        }
    }

    private class Frame(private val max: Float) : ViewOutlineProvider() {
        var pill = false
        var measurePending = false
        var measured: Rect? = null
        var measuredW = 0
        var measuredH = 0
        val content = Rect()

        fun radius(): Float {
            val half = min(content.width(), content.height()) / 2f
            return if (pill) half else min(max, half)
        }

        override fun getOutline(view: View, outline: Outline) {
            if (content.isEmpty) content.set(0, 0, view.width, view.height)
            outline.setRoundRect(content, radius())
        }
    }

    /** A 1dp translucent white line inside the frame's edge. */
    /** The same depth ring as the interface's buttons, on the widget's visible card. */
    private class Ring(private val frame: Frame, density: Float, private val dark: () -> Boolean) : Drawable() {
        private val painter = DepthRingPainter(density)
        private val rect = RectF()

        override fun draw(canvas: Canvas) {
            val c = if (frame.content.isEmpty) bounds else frame.content
            rect.set(c)
            painter.draw(canvas, rect, frame.radius(), dark())
        }

        override fun setAlpha(alpha: Int) = Unit

        override fun setColorFilter(colorFilter: ColorFilter?) = Unit

        @Deprecated("Deprecated in Java")
        override fun getOpacity() = PixelFormat.TRANSLUCENT
    }
}
