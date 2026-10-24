package io.github.jacomartins.canyonui.model

/** A cell position on a grid, in columns ([x]) and rows ([y]). */
data class Cell(val x: Int, val y: Int)

/**
 * Tracks which cells of a [columns] x [rows] grid are taken. Pure Kotlin so placement and
 * collision rules can be unit-tested without a device.
 */
class GridOccupancy(val columns: Int, val rows: Int) {

    init {
        require(columns > 0 && rows > 0) { "Grid must be at least 1x1, was ${columns}x$rows" }
    }

    private val taken = BooleanArray(columns * rows)

    /** True when the whole span fits inside the grid. */
    fun fits(x: Int, y: Int, spanX: Int = 1, spanY: Int = 1): Boolean =
        x >= 0 && y >= 0 && spanX > 0 && spanY > 0 && x + spanX <= columns && y + spanY <= rows

    /** True when the span fits and none of its cells are taken. */
    fun isFree(x: Int, y: Int, spanX: Int = 1, spanY: Int = 1): Boolean {
        if (!fits(x, y, spanX, spanY)) return false
        for (cy in y until y + spanY) {
            for (cx in x until x + spanX) {
                if (taken[cy * columns + cx]) return false
            }
        }
        return true
    }

    /** Marks a span as taken. Returns false (and changes nothing) if it does not fit. */
    fun mark(x: Int, y: Int, spanX: Int = 1, spanY: Int = 1): Boolean = set(x, y, spanX, spanY, true)

    fun clear(x: Int, y: Int, spanX: Int = 1, spanY: Int = 1): Boolean = set(x, y, spanX, spanY, false)

    private fun set(x: Int, y: Int, spanX: Int, spanY: Int, value: Boolean): Boolean {
        if (!fits(x, y, spanX, spanY)) return false
        for (cy in y until y + spanY) {
            for (cx in x until x + spanX) {
                taken[cy * columns + cx] = value
            }
        }
        return true
    }

    /** First free position for the span, scanning row by row from the top-left; null if full. */
    fun findFirstFree(spanX: Int = 1, spanY: Int = 1): Cell? {
        for (y in 0..rows - spanY) {
            for (x in 0..columns - spanX) {
                if (isFree(x, y, spanX, spanY)) return Cell(x, y)
            }
        }
        return null
    }

    fun freeCellCount(): Int = taken.count { !it }

    /**
     * The free position for the span whose centre is closest to the centre of the span at
     * ([x], [y]); null if nothing fits. Used to snap a drop next to where the finger is.
     */
    fun findNearestFree(x: Int, y: Int, spanX: Int = 1, spanY: Int = 1): Cell? {
        var best: Cell? = null
        var bestDist = Int.MAX_VALUE
        for (cy in 0..rows - spanY) {
            for (cx in 0..columns - spanX) {
                if (!isFree(cx, cy, spanX, spanY)) continue
                val dx = cx - x
                val dy = cy - y
                val d = dx * dx + dy * dy
                if (d < bestDist) {
                    bestDist = d
                    best = Cell(cx, cy)
                }
            }
        }
        return best
    }
}
