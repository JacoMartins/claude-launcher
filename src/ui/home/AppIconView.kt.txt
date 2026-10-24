package io.github.jacomartins.canyonui.ui.home

import android.annotation.SuppressLint
import android.content.Context
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.Paint
import android.graphics.Rect
import android.text.TextPaint
import android.text.TextUtils
import android.util.TypedValue
import android.view.View
import android.view.animation.DecelerateInterpolator
import io.github.jacomartins.canyonui.data.AppInfo
import io.github.jacomartins.canyonui.util.Motion

/**
 * One app on the home screen or dock: icon plus a one-line label, drawn directly (no child
 * views, no per-frame allocation). The label is ellipsized once per size or text change.
 */
@SuppressLint("ViewConstructor")
class AppIconView(context: Context, private var style: LabelStyle) : View(context) {

    /** Colours for labels drawn over the wallpaper. */
    class LabelStyle(
        var color: Int,
        var shadowColor: Int,
        var showLabel: Boolean = true,
        var iconCellFraction: Float = 0.68f,
        var maxIconDp: Int = 64,
    )

    var app: AppInfo? = null
        private set

    private var icon: Bitmap? = null
    private var label: String = ""
    private var shownLabel: String = ""

    private val dm = resources.displayMetrics
    private val gapPx = LABEL_GAP_DP * dm.density
    private val iconRect = Rect()
    private val bitmapPaint = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG)
    private val textPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
        textSize = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, LABEL_SP, dm)
        textAlign = Paint.Align.CENTER
    }
    private val pressInterpolator = DecelerateInterpolator()
    private val badgePainter = BadgePainter()

    /** Notifications waiting for this app (0 = no badge); [badgeNumbers] false shows a dot. */
    var badge = 0
        set(value) { if (field != value) { field = value; invalidate() } }
    var badgeNumbers = true
        set(value) { if (field != value) { field = value; invalidate() } }

    init {
        isClickable = true
        isLongClickable = true
        isFocusable = true
        applyStyle()
    }

    fun bind(info: AppInfo) {
        if (app === info) return
        app = info
        icon = info.icon
        label = info.label
        contentDescription = info.label
        updateShownLabel()
        invalidate()
    }

    /** Home and dock have separate styles; an icon takes the style of where it is placed. */
    fun useStyle(newStyle: LabelStyle) {
        if (newStyle === style) return
        style = newStyle
        applyStyle()
    }

    /** Re-reads [style] (label colour changes with the wallpaper). */
    fun applyStyle() {
        textPaint.color = style.color
        val shadowRadius = if (style.shadowColor == 0) 0f else 3f * dm.density
        textPaint.setShadowLayer(shadowRadius, 0f, 1f * dm.density, style.shadowColor)
        invalidate()
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        updateShownLabel()
    }

    private fun updateShownLabel() {
        val avail = (width - 4 * dm.density).coerceAtLeast(0f)
        shownLabel = if (avail <= 0f) label else TextUtils.ellipsize(label, textPaint, avail, TextUtils.TruncateAt.END).toString()
    }

    /** Where the icon is drawn, in this view's coordinates (also used for the drag image). */
    fun iconBounds(out: Rect): Rect {
        val textH = if (style.showLabel) textPaint.fontSpacing else 0f
        val maxIconPx = (style.maxIconDp * dm.density).toInt()
        val iconPx = minOf(maxIconPx, (width * style.iconCellFraction).toInt(), (height - textH - gapPx).toInt()).coerceAtLeast(0)
        val blockH = iconPx + if (style.showLabel) gapPx + textH else 0f
        val top = ((height - blockH) / 2f).toInt()
        val left = (width - iconPx) / 2
        out.set(left, top, left + iconPx, top + iconPx)
        return out
    }

    override fun onDraw(canvas: Canvas) {
        val bmp = icon ?: return
        iconBounds(iconRect)
        val iconPx = iconRect.width()
        val top = iconRect.top
        // Pressed: a touch brighter, with the press scale.
        bitmapPaint.colorFilter = if (isPressed) PRESSED_FILTER else null
        canvas.drawBitmap(bmp, null, iconRect, bitmapPaint)
        badgePainter.draw(canvas, iconRect, badge, badgeNumbers)
        if (style.showLabel) {
            val baseline = top + iconPx + gapPx - textPaint.ascent() // ascent() does not allocate
            canvas.drawText(shownLabel, width / 2f, baseline, textPaint)
        }
    }

    companion object {
        const val LABEL_SP = 11f

        private val PRESSED_FILTER = android.graphics.ColorMatrixColorFilter(
            android.graphics.ColorMatrix().apply { setScale(1.12f, 1.12f, 1.12f, 1f) },
        )
        const val LABEL_GAP_DP = 3f
    }

    override fun setPressed(pressed: Boolean) {
        if (pressed == isPressed) return
        super.setPressed(pressed)
        invalidate()
        val target = if (pressed) Motion.PRESS_SCALE else 1f
        if (Motion.enabled()) {
            animate().scaleX(target).scaleY(target).setDuration(Motion.PRESS_DURATION_MS)
                .setInterpolator(pressInterpolator).start()
        } else {
            scaleX = target
            scaleY = target
        }
    }
}
