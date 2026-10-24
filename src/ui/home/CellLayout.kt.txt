package io.github.jacomartins.canyonui.ui.home

import android.annotation.SuppressLint
import android.content.Context
import android.view.ViewGroup
import androidx.core.view.isGone

/**
 * A fixed grid of [columns] x [rows] cells. Children carry their cell in [LayoutParams] and are
 * measured to exactly their span. Columns are mirrored in right-to-left layouts.
 */
@SuppressLint("ViewConstructor") // Created in code only; never inflated from XML.
class CellLayout(context: Context, columns: Int, rows: Int) : ViewGroup(context) {

    var columns = columns
        private set
    var rows = rows
        private set

    /**
     * [inset]: pixels kept free on every side inside the cells (widgets keep a gap between them).
     * [maxThickness]: when above 0, a child one cell wide or one row tall is at most this thick
     * across that cell, centred in it (pill widgets).
     */
    class LayoutParams(
        var cellX: Int,
        var cellY: Int,
        var spanX: Int = 1,
        var spanY: Int = 1,
        var inset: Int = 0,
        var maxThickness: Int = 0,
    ) : ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT)

    /** Size of a child's area along one axis, and its offset inside its cells. */
    private fun extent(cells: Int, cell: Int, lp: LayoutParams): Int {
        val full = (cell * cells - 2 * lp.inset).coerceAtLeast(0)
        return if (cells == 1 && lp.maxThickness > 0) minOf(full, lp.maxThickness) else full
    }

    private val d = resources.displayMetrics.density
    private val hintRect = android.graphics.RectF()
    private val hintPaint = android.graphics.Paint(android.graphics.Paint.ANTI_ALIAS_FLAG)
    private val ring = io.github.jacomartins.canyonui.ui.design.DepthRingPainter(resources.displayMetrics.density)
    /** Light hints mean a dark wallpaper behind the page: the ring takes its dark-surface values. */
    private var darkBehind = true
    private var hintVisible = false
    private var hintFolder = false
    private var hintX = 0
    private var hintY = 0
    private var hintSpanX = 1
    private var hintSpanY = 1

    init {
        setWillNotDraw(false)
    }


    private val plusPaint = android.graphics.Paint(android.graphics.Paint.ANTI_ALIAS_FLAG).apply {
        style = android.graphics.Paint.Style.STROKE
        strokeCap = android.graphics.Paint.Cap.ROUND
    }

    /** Edit mode's last page: an empty page with a plus; tapping it adds a page. */
    var addPagePlaceholder = false
        set(value) {
            if (field != value) {
                field = value
                contentDescription = if (value) resources.getString(io.github.jacomartins.canyonui.R.string.add_page) else null
                invalidate()
            }
        }

    /** Edit mode: a rounded translucent card behind the page shows its bounds. */
    var showBackdrop = false
        set(value) {
            if (field != value) { field = value; invalidate() }
        }

    fun setColors(hint: Int, backdrop: Int) {
        hintPaint.color = hint
        // The plus in the hint's hue, fully opaque: white over dark wallpapers, black over light.
        plusPaint.color = hint or 0xFF000000.toInt()
        darkBehind = (hint and 0xFF) > 0x80
        invalidate()
    }

    /**
     * Highlights where a dragged item will land. [folder] marks a drop that makes or fills a
     * folder: the highlight is the icon-sized square of the target instead of the cell.
     */
    fun showDropHint(x: Int, y: Int, spanX: Int, spanY: Int, folder: Boolean) {
        if (hintVisible && x == hintX && y == hintY && spanX == hintSpanX && spanY == hintSpanY && folder == hintFolder) return
        hintVisible = true
        hintX = x; hintY = y; hintSpanX = spanX; hintSpanY = spanY; hintFolder = folder
        invalidate()
    }

    fun clearDropHint() {
        if (!hintVisible) return
        hintVisible = false
        invalidate()
    }

    /** Cell under a point in this view's coordinates, or null outside the grid. */
    fun cellAt(x: Float, y: Float, out: IntArray): Boolean {
        val cw = cellWidth
        val ch = cellHeight
        if (cw <= 0 || ch <= 0) return false
        var col = ((x - paddingLeft) / cw).toInt()
        val row = ((y - paddingTop) / ch).toInt()
        if (x < paddingLeft || y < paddingTop || col !in 0 until columns || row !in 0 until rows) return false
        if (layoutDirection == LAYOUT_DIRECTION_RTL) col = columns - 1 - col
        out[0] = col
        out[1] = row
        return true
    }

    /** Left/top of a cell in this view's coordinates (RTL-aware). */
    fun cellLeft(col: Int, spanX: Int = 1): Int {
        val c = if (layoutDirection == LAYOUT_DIRECTION_RTL) columns - col - spanX else col
        return paddingLeft + c * cellWidth
    }

    fun cellTop(row: Int): Int = paddingTop + row * cellHeight

    override fun onDraw(canvas: android.graphics.Canvas) {
        if (showBackdrop) {
            val r = 28 * d
            hintRect.set(4 * d, 0f, width - 4 * d, height.toFloat())
            // The page itself is clear (the effect is edit mode's background behind it); only
            // OneDepth's border, the same as the interface's buttons, outlines it.
            ring.draw(canvas, hintRect, r, darkBehind)
        }
        if (addPagePlaceholder) {
            val cx = width / 2f
            val cy = height / 2f
            val arm = 18 * d
            plusPaint.strokeWidth = 4 * d
            canvas.drawLine(cx - arm, cy, cx + arm, cy, plusPaint)
            canvas.drawLine(cx, cy - arm, cx, cy + arm, plusPaint)
        }
        if (hintVisible) {
            val cw = cellWidth
            val ch = cellHeight
            val left = cellLeft(hintX, hintSpanX).toFloat()
            val top = cellTop(hintY).toFloat()
            if (hintFolder) {
                val size = minOf(cw, ch) * 0.86f
                val cx = left + cw / 2f
                val cy = top + ch / 2f
                hintRect.set(cx - size / 2, cy - size / 2, cx + size / 2, cy + size / 2)
                canvas.drawRoundRect(hintRect, size * 0.3f, size * 0.3f, hintPaint)
            } else {
                val inset = 4 * d
                hintRect.set(left + inset, top + inset, left + cw * hintSpanX - inset, top + ch * hintSpanY - inset)
                canvas.drawRoundRect(hintRect, 20 * d, 20 * d, hintPaint)
            }
        }
    }

    fun setGridSize(columns: Int, rows: Int) {
        if (columns == this.columns && rows == this.rows) return
        this.columns = columns
        this.rows = rows
        requestLayout()
    }

    val cellWidth: Int get() = (width - paddingLeft - paddingRight) / columns
    val cellHeight: Int get() = (height - paddingTop - paddingBottom) / rows

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val w = MeasureSpec.getSize(widthMeasureSpec)
        val h = MeasureSpec.getSize(heightMeasureSpec)
        setMeasuredDimension(w, h)
        val cw = (w - paddingLeft - paddingRight) / columns
        val ch = (h - paddingTop - paddingBottom) / rows
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            if (child.isGone) continue
            val lp = child.layoutParams as LayoutParams
            child.measure(
                MeasureSpec.makeMeasureSpec(extent(lp.spanX, cw, lp), MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(extent(lp.spanY, ch, lp), MeasureSpec.EXACTLY),
            )
        }
    }

    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        val cw = cellWidth
        val ch = cellHeight
        val rtl = layoutDirection == LAYOUT_DIRECTION_RTL
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            if (child.isGone) continue
            val lp = child.layoutParams as LayoutParams
            val col = if (rtl) columns - lp.cellX - lp.spanX else lp.cellX
            // Centred in its cells (pills too), so the gap above and below matches on every row.
            val left = paddingLeft + col * cw + (cw * lp.spanX - child.measuredWidth) / 2
            val freeY = ch * lp.spanY - child.measuredHeight
            val top = paddingTop + lp.cellY * ch + freeY / 2
            child.layout(left, top, left + child.measuredWidth, top + child.measuredHeight)
        }
    }

    override fun checkLayoutParams(p: ViewGroup.LayoutParams?) = p is LayoutParams
    override fun generateDefaultLayoutParams(): ViewGroup.LayoutParams = LayoutParams(0, 0)
    override fun shouldDelayChildPressedState() = false

}
