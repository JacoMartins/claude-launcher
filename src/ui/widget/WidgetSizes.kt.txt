package io.github.jacomartins.canyonui.ui.widget

import android.appwidget.AppWidgetHostView
import android.appwidget.AppWidgetProviderInfo
import android.os.Build
import android.os.Bundle
import android.util.SizeF
import kotlin.math.ceil
import kotlin.math.max

/** A widget's size range in cells. */
data class SpanRange(val minX: Int, val minY: Int, val maxX: Int, val maxY: Int) {
    val resizable get() = minX < maxX || minY < maxY

    fun intersect(o: SpanRange) = SpanRange(max(minX, o.minX), max(minY, o.minY), minOf(maxX, o.maxX), minOf(maxY, o.maxY))
}

/**
 * Converts between a provider's declared sizes (pixels, already scaled to this screen) and grid
 * cells of [cellWidth] x [cellHeight] pixels.
 */
class WidgetSizes(
    private val cellWidth: Int,
    private val cellHeight: Int,
    private val columns: Int,
    private val rows: Int,
    private val density: Float,
    /** The "Resize any widget" setting: every widget may take any size from 1x1 to the page. */
    private val anySize: Boolean = false,
    /** Space kept free around a widget inside its cells, per side. */
    private val insetPx: Int = 0,
    /** Widgets one cell wide or one row tall are at most this thick (pills); 0 = no cap. */
    private val pillMaxPx: Int = 0,
    /** Report the whole cell to pill widgets instead of the pill's thickness (see settings). */
    private val reportFullCell: Boolean = false,
) {
    private fun extentPx(spans: Int, cell: Int): Int {
        val full = spans * cell - 2 * insetPx
        return if (spans == 1 && pillMaxPx > 0 && !reportFullCell) minOf(full, pillMaxPx) else full
    }

    private fun widthDp(spanX: Int) = extentPx(spanX, cellWidth) / density
    private fun heightDp(spanY: Int) = extentPx(spanY, cellHeight) / density


    private fun spansFor(px: Int, cell: Int, limit: Int) = ceil(px.toFloat() / cell.coerceAtLeast(1)).toInt().coerceIn(1, limit)

    private fun minSpans(px: Int, cell: Int) = (px / cell.coerceAtLeast(1)).coerceAtLeast(1)

    /** The size a widget starts at. */
    fun defaultSpans(info: AppWidgetProviderInfo): Pair<Int, Int> {
        if (Build.VERSION.SDK_INT >= 31 && info.targetCellWidth > 0 && info.targetCellHeight > 0) {
            return info.targetCellWidth.coerceIn(1, columns) to info.targetCellHeight.coerceIn(1, rows)
        }
        return spansFor(info.minWidth, cellWidth, columns) to spansFor(info.minHeight, cellHeight, rows)
    }

    /** How small and large the user may make it; fixed axes stay at the default size. */
    fun range(info: AppWidgetProviderInfo): SpanRange {
        if (anySize) return SpanRange(1, 1, columns, rows)
        val (dx, dy) = defaultSpans(info)
        val horizontal = info.resizeMode and AppWidgetProviderInfo.RESIZE_HORIZONTAL != 0
        val vertical = info.resizeMode and AppWidgetProviderInfo.RESIZE_VERTICAL != 0
        var maxX = columns
        var maxY = rows
        if (Build.VERSION.SDK_INT >= 31) {
            if (info.maxResizeWidth > 0) maxX = spansFor(info.maxResizeWidth, cellWidth, columns)
            if (info.maxResizeHeight > 0) maxY = spansFor(info.maxResizeHeight, cellHeight, rows)
        }
        // The smallest size rounds down: providers state minimums for the guideline cell, which is
        // a little smaller than a cell here, and their layouts (Android 12 responsive sizes) adapt.
        // Rounding up kept widgets like a clock or a weather pill from going to 1 cell.
        val minX = if (horizontal) minSpans(info.minResizeWidth.takeIf { it > 0 } ?: info.minWidth, cellWidth).coerceAtMost(dx) else dx
        val minY = if (vertical) minSpans(info.minResizeHeight.takeIf { it > 0 } ?: info.minHeight, cellHeight).coerceAtMost(dy) else dy
        return SpanRange(
            minX, minY,
            if (horizontal) maxOf(maxX, dx) else dx,
            if (vertical) maxOf(maxY, dy) else dy,
        )
    }

    /**
     * Tells the provider the size it is shown at, in dp, so it can pick a matching layout.
     * [night] is passed along as well: a change of it is an options change, which makes widgets
     * that pick their colours when they update (rather than from the theme) draw again after a
     * switch between light and dark mode.
     */
    fun applySize(view: AppWidgetHostView, spanX: Int, spanY: Int, night: Boolean) {
        val w = widthDp(spanX)
        val h = heightDp(spanY)
        val extras = Bundle().apply { putBoolean(NIGHT_OPTION, night) }
        try {
            if (Build.VERSION.SDK_INT >= 31) {
                view.updateAppWidgetSize(extras, listOf(SizeF(w, h)))
            } else {
                @Suppress("DEPRECATION")
                view.updateAppWidgetSize(extras, w.toInt(), h.toInt(), w.toInt(), h.toInt())
            }
        } catch (e: RuntimeException) {
            // The provider may be gone or updating; it gets the size on its next bind.
        }
    }

    private companion object {
        /** Canyon UI's own option key: the dark mode the widget is shown in. */
        const val NIGHT_OPTION = "io.github.jacomartins.canyonui.NIGHT"
    }

    /** Options for binding a new widget at [spanX] x [spanY]. */
    fun options(spanX: Int, spanY: Int): Bundle {
        val w = widthDp(spanX).toInt()
        val h = heightDp(spanY).toInt()
        return Bundle().apply {
            putInt(android.appwidget.AppWidgetManager.OPTION_APPWIDGET_MIN_WIDTH, w)
            putInt(android.appwidget.AppWidgetManager.OPTION_APPWIDGET_MIN_HEIGHT, h)
            putInt(android.appwidget.AppWidgetManager.OPTION_APPWIDGET_MAX_WIDTH, w)
            putInt(android.appwidget.AppWidgetManager.OPTION_APPWIDGET_MAX_HEIGHT, h)
            if (Build.VERSION.SDK_INT >= 31) {
                putParcelableArrayList(android.appwidget.AppWidgetManager.OPTION_APPWIDGET_SIZES, arrayListOf(SizeF(w.toFloat(), h.toFloat())))
            }
        }
    }
}
