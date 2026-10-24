package io.github.jacomartins.canyonui.ui.widget

import android.annotation.SuppressLint
import android.content.Context
import android.graphics.Canvas
import android.graphics.Paint
import android.graphics.RectF
import android.view.HapticFeedbackConstants
import android.view.MotionEvent
import android.view.View
import io.github.jacomartins.canyonui.model.GridOccupancy
import io.github.jacomartins.canyonui.ui.home.CellLayout
import io.github.jacomartins.canyonui.ui.home.ViewMap
import kotlin.math.abs
import kotlin.math.roundToInt

/**
 * A frame with handles around a widget being resized. It covers the whole root view but only
 * takes touches that start on a handle; any other touch dismisses it and passes through.
 *
 * Dragging a handle snaps the widget to whole cells as the finger crosses cell midpoints, within
 * the widget's [range], inside the page, and never over another item ([occupied] holds every
 * other item on the page). The widget's own view is moved live; [onResize] gets the final cells.
 */
@SuppressLint("ViewConstructor")
class ResizeFrame(
    context: Context,
    private val root: View,
    private val page: CellLayout,
    private val target: View,
    private val occupied: GridOccupancy,
    private val range: SpanRange,
    private val onStart: () -> Unit,
    private val onResize: (x: Int, y: Int, spanX: Int, spanY: Int) -> Unit,
    private val onDismiss: () -> Unit,
) : View(context) {

    private enum class Edge { LEFT, TOP, RIGHT, BOTTOM }

    private val d = resources.displayMetrics.density
    private val stroke = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 2 * d
        color = 0xFFFFFFFF.toInt()
        setShadowLayer(3 * d, 0f, 0f, 0x55000000)
    }
    private val handleFill = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = 0xFFFFFFFF.toInt()
        setShadowLayer(3 * d, 0f, d, 0x55000000)
    }
    private val frame = RectF()
    private val pt = FloatArray(2)
    private val handleRadius = 7 * d
    private val touchRadius = 28 * d

    private val lp = target.layoutParams as CellLayout.LayoutParams
    // The widget's cells when the current handle drag began.
    private var startX = lp.cellX
    private var startY = lp.cellY
    private var startSpanX = lp.spanX
    private var startSpanY = lp.spanY
    private var edge: Edge? = null
    private var downX = 0f
    private var downY = 0f

    init {
        importantForAccessibility = IMPORTANT_FOR_ACCESSIBILITY_NO
    }

    private val horizontal get() = range.minX < range.maxX
    private val vertical get() = range.minY < range.maxY

    /** The widget's current cell rectangle in root coordinates. */
    private fun updateFrame() {
        val left = page.cellLeft(lp.cellX, lp.spanX).toFloat()
        val top = page.cellTop(lp.cellY).toFloat()
        pt[0] = left; pt[1] = top
        ViewMap.localToRoot(root, page, pt)
        val l = pt[0]
        val t = pt[1]
        pt[0] = left + page.cellWidth * lp.spanX; pt[1] = top + page.cellHeight * lp.spanY
        ViewMap.localToRoot(root, page, pt)
        val inset = 2 * d
        frame.set(l + inset, t + inset, pt[0] - inset, pt[1] - inset)
    }

    private fun handles(): List<Pair<Edge, Pair<Float, Float>>> = buildList {
        if (horizontal) {
            add(Edge.LEFT to (frame.left to frame.centerY()))
            add(Edge.RIGHT to (frame.right to frame.centerY()))
        }
        if (vertical) {
            add(Edge.TOP to (frame.centerX() to frame.top))
            add(Edge.BOTTOM to (frame.centerX() to frame.bottom))
        }
    }

    override fun onDraw(canvas: Canvas) {
        updateFrame()
        val r = 22 * d
        canvas.drawRoundRect(frame, r, r, stroke)
        for ((_, p) in handles()) canvas.drawCircle(p.first, p.second, handleRadius, handleFill)
    }

    @SuppressLint("ClickableViewAccessibility") // Resizing is also offered as a menu action.
    override fun onTouchEvent(ev: MotionEvent): Boolean {
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                updateFrame()
                edge = handles().firstOrNull { (_, p) -> abs(ev.x - p.first) < touchRadius && abs(ev.y - p.second) < touchRadius }?.first
                if (edge == null) {
                    onDismiss()
                    return false
                }
                downX = ev.x; downY = ev.y
                startX = lp.cellX; startY = lp.cellY; startSpanX = lp.spanX; startSpanY = lp.spanY
                parent?.requestDisallowInterceptTouchEvent(true)
                onStart()
                return true
            }
            MotionEvent.ACTION_MOVE -> edge?.let { drag(it, ev.x - downX, ev.y - downY) }
            MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> {
                edge = null
                if (lp.cellX != startX || lp.cellY != startY || lp.spanX != startSpanX || lp.spanY != startSpanY) {
                    onResize(lp.cellX, lp.cellY, lp.spanX, lp.spanY)
                }
            }
        }
        return true
    }

    private fun drag(e: Edge, dxRoot: Float, dyRoot: Float) {
        val scale = ViewMap.scaleOf(root, page).takeIf { it > 0f } ?: 1f
        val rtl = page.layoutDirection == LAYOUT_DIRECTION_RTL
        // Cells moved, in screen direction; columns are mirrored in RTL.
        val dCol = (dxRoot / scale / page.cellWidth).roundToInt().let { if (rtl) -it else it }
        val dRow = (dyRoot / scale / page.cellHeight).roundToInt()
        var x = startX
        var y = startY
        var sx = startSpanX
        var sy = startSpanY
        val growsTowardLowX = (e == Edge.LEFT) != rtl
        when (e) {
            Edge.LEFT, Edge.RIGHT -> {
                val delta = if (growsTowardLowX) -dCol else dCol
                sx = (startSpanX + delta).coerceIn(range.minX, range.maxX)
                if (growsTowardLowX) x = startX + startSpanX - sx
            }
            Edge.TOP -> {
                sy = (startSpanY - dRow).coerceIn(range.minY, range.maxY)
                y = startY + startSpanY - sy
            }
            Edge.BOTTOM -> sy = (startSpanY + dRow).coerceIn(range.minY, range.maxY)
        }
        // Back off one cell at a time until it fits inside the page without overlapping.
        while (!fits(x, y, sx, sy)) {
            when (e) {
                Edge.LEFT, Edge.RIGHT -> {
                    if (sx == startSpanX) return
                    if (sx > startSpanX) sx-- else sx++
                    if (growsTowardLowX) x = startX + startSpanX - sx
                }
                Edge.TOP -> {
                    if (sy == startSpanY) return
                    if (sy > startSpanY) sy-- else sy++
                    y = startY + startSpanY - sy
                }
                Edge.BOTTOM -> {
                    if (sy == startSpanY) return
                    if (sy > startSpanY) sy-- else sy++
                }
            }
        }
        if (x != lp.cellX || y != lp.cellY || sx != lp.spanX || sy != lp.spanY) {
            lp.cellX = x; lp.cellY = y; lp.spanX = sx; lp.spanY = sy
            target.requestLayout()
            performHapticFeedback(HapticFeedbackConstants.CLOCK_TICK)
            invalidate()
        }
    }

    private fun fits(x: Int, y: Int, sx: Int, sy: Int) =
        x >= 0 && y >= 0 && x + sx <= page.columns && y + sy <= page.rows && occupied.isFree(x, y, sx, sy)

    /** Redraws after the page moved or scaled (edit mode zoom). */
    fun refresh() = invalidate()
}
