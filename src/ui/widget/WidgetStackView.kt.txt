package io.github.jacomartins.canyonui.ui.widget

import android.animation.ValueAnimator
import android.annotation.SuppressLint
import android.content.Context
import android.graphics.Canvas
import android.graphics.Paint
import android.graphics.drawable.GradientDrawable
import android.view.Gravity
import android.view.MotionEvent
import android.view.VelocityTracker
import android.view.View
import android.view.ViewConfiguration
import android.view.animation.DecelerateInterpolator
import android.widget.FrameLayout
import android.widget.TextView
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.util.Motion
import kotlin.math.abs

/**
 * Widgets of one size in one place. A horizontal swipe moves to the next or previous widget
 * (wrapping around); dots at the bottom show which one is in front while the stack is touched.
 * Only the front widget and, during a swipe, its neighbour are visible.
 */
@SuppressLint("ViewConstructor")
class WidgetStackView(context: Context) : FrameLayout(context) {

    /** The stack's widgets, front first, in stack order. */
    private val pages = ArrayList<View>()
    var current = 0
        private set

    private val d = resources.displayMetrics.density
    private val slop = ViewConfiguration.get(context).scaledTouchSlop
    private val minFling = ViewConfiguration.get(context).scaledMinimumFlingVelocity * 4f
    private var downX = 0f
    private var downY = 0f
    private var lastX = 0f
    private var swiping = false
    private var offset = 0f
    private var tracker: VelocityTracker? = null
    private var settle: ValueAnimator? = null
    private val decelerate = DecelerateInterpolator()

    private val dotPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private var dotsAlpha = 0f
    private val hideDots = Runnable { animateDots(0f) }

    var onPageChanged: ((Int) -> Unit)? = null

    /**
     * Slideshow: above 0, the stack turns to its next widget every this many milliseconds. The
     * page follows the clock (how many periods have passed), not how long the home screen has
     * been showing, so coming back from an app shows the page it would be on by now.
     */
    var slideshowMs = 0L
        set(value) {
            if (field == value) return
            field = value
            scheduleSlide(animate = false)
        }

    private val slide = Runnable { scheduleSlide(animate = true) }

    private fun scheduleSlide(animate: Boolean) {
        removeCallbacks(slide)
        val period = slideshowMs
        if (period <= 0 || pages.size < 2 || !isAttachedToWindow) return
        val now = System.currentTimeMillis()
        val due = ((now / period) % pages.size).toInt()
        if (due != current && !swiping && settle?.isRunning != true) {
            if (animate && isShown && due == index(current + 1)) {
                settleTo(1)
            } else {
                current = due
                offset = 0f
                layoutPages()
                onPageChanged?.invoke(current)
            }
        }
        postDelayed(slide, period - now % period + 5)
    }

    override fun onAttachedToWindow() {
        super.onAttachedToWindow()
        scheduleSlide(animate = false)
    }

    override fun onWindowVisibilityChanged(visibility: Int) {
        super.onWindowVisibilityChanged(visibility)
        // Back on screen: catch up at once (no animation), then follow the clock again.
        if (visibility == VISIBLE) scheduleSlide(animate = false) else removeCallbacks(slide)
    }

    init {
        clipChildren = true
        setWillNotDraw(false)
        io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.track(this)
    }

    private val framePath = android.graphics.Path()

    /** Puts [views] in the stack (keeping the current page when it is still there). */
    fun setPages(views: List<View>) {
        val front = pages.getOrNull(current)
        if (views == pages) return
        for (v in pages) if (v !in views) removeView(v)
        pages.clear()
        pages += views
        for (v in views) if (v.parent !== this) {
            (v.parent as? android.view.ViewGroup)?.removeView(v)
            addView(v, LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT))
        }
        current = views.indexOf(front).takeIf { it >= 0 } ?: 0
        offset = 0f
        layoutPages()
    }

    val pageCount get() = pages.size

    fun frontView(): View? = pages.getOrNull(current)

    private fun index(i: Int) = ((i % pages.size) + pages.size) % pages.size

    private fun layoutPages() {
        val w = width.toFloat()
        for ((i, v) in pages.withIndex()) {
            val rel = when (i) {
                current -> 0
                index(current + 1) -> if (offset < 0) 1 else Int.MAX_VALUE
                index(current - 1) -> if (offset > 0) -1 else Int.MAX_VALUE
                else -> Int.MAX_VALUE
            }
            if (rel == Int.MAX_VALUE) {
                v.visibility = INVISIBLE
            } else {
                v.visibility = VISIBLE
                v.translationX = rel * w + offset
            }
        }
    }

    override fun onLayout(changed: Boolean, left: Int, top: Int, right: Int, bottom: Int) {
        super.onLayout(changed, left, top, right, bottom)
        layoutPages()
    }

    override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
        if (ev.actionMasked == MotionEvent.ACTION_DOWN && pages.size > 1) {
            // Keep the page swipe and the drawer gesture away until the direction is known.
            parent?.requestDisallowInterceptTouchEvent(true)
            removeCallbacks(hideDots)
            animateDots(1f)
        }
        if (ev.actionMasked == MotionEvent.ACTION_UP || ev.actionMasked == MotionEvent.ACTION_CANCEL) {
            postDelayed(hideDots, DOTS_MS)
        }
        return super.dispatchTouchEvent(ev)
    }

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        if (pages.size < 2) return false
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                downX = ev.x; downY = ev.y; lastX = ev.x
                swiping = false
                settle?.cancel()
                tracker?.clear()
                track(ev)
            }
            MotionEvent.ACTION_MOVE -> {
                track(ev)
                val dx = ev.x - downX
                val dy = ev.y - downY
                if (abs(dy) > slop && abs(dy) > abs(dx)) {
                    // Vertical: hand the gesture back to the parents (drawer, scrolling).
                    parent?.requestDisallowInterceptTouchEvent(false)
                } else if (abs(dx) > slop && abs(dx) > abs(dy)) {
                    swiping = true
                    lastX = ev.x
                    pages.forEach { it.cancelLongPress() }
                    return true
                }
            }
        }
        return false
    }

    @SuppressLint("ClickableViewAccessibility") // Swipes only; widgets inside stay accessible.
    override fun onTouchEvent(ev: MotionEvent): Boolean {
        if (!swiping) return pages.size > 1
        track(ev)
        when (ev.actionMasked) {
            MotionEvent.ACTION_MOVE -> {
                offset += ev.x - lastX
                lastX = ev.x
                layoutPages()
            }
            MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> {
                tracker?.computeCurrentVelocity(1000)
                val vx = tracker?.xVelocity ?: 0f
                val w = width.toFloat()
                val dir = when {
                    abs(vx) > minFling -> if (vx < 0) 1 else -1
                    abs(offset) > w / 3 -> if (offset < 0) 1 else -1
                    else -> 0
                }
                swiping = false
                settleTo(dir)
            }
        }
        return true
    }

    private fun settleTo(dir: Int) {
        val w = width.toFloat()
        val target = -dir * w
        val end = {
            if (dir != 0) {
                current = index(current + dir)
                onPageChanged?.invoke(current)
                announceForAccessibility(pages[current].contentDescription ?: "")
            }
            offset = 0f
            layoutPages()
            invalidate()
        }
        if (!Motion.enabled()) { end(); return }
        settle = ValueAnimator.ofFloat(offset, target).apply {
            duration = 220
            interpolator = decelerate
            addUpdateListener { offset = it.animatedValue as Float; layoutPages() }
            addListener(object : android.animation.AnimatorListenerAdapter() {
                override fun onAnimationEnd(animation: android.animation.Animator) = end()
            })
            start()
        }
    }

    private fun track(ev: MotionEvent) {
        val t = tracker ?: VelocityTracker.obtain().also { tracker = it }
        t.addMovement(ev)
    }

    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        tracker?.recycle()
        tracker = null
        settle?.cancel()
        removeCallbacks(hideDots)
        removeCallbacks(slide)
    }

    private fun animateDots(to: Float) {
        if (!Motion.enabled()) { dotsAlpha = to; invalidate(); return }
        ValueAnimator.ofFloat(dotsAlpha, to).apply {
            duration = 160
            addUpdateListener { dotsAlpha = it.animatedValue as Float; invalidate() }
            start()
        }
    }

    override fun dispatchDraw(canvas: Canvas) {
        // Frosted and glass behind the whole stack (its widgets are plain inside it), so a
        // see-through widget keeps the effect as the slideshow turns.
        val frameRadius = WidgetFrame.framePath(this, framePath)
        if (frameRadius >= 0f) io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.draw(canvas, this, framePath, frameRadius)
        super.dispatchDraw(canvas)
        if (pages.size < 2 || dotsAlpha <= 0f) return
        val r = 3 * d
        val gap = 10 * d
        val total = (pages.size - 1) * gap
        var x = width / 2f - total / 2f
        val y = height - 8 * d
        for (i in pages.indices) {
            dotPaint.color = if (i == current) 0xFFFFFFFF.toInt() else 0x80FFFFFF.toInt()
            dotPaint.alpha = (dotPaint.alpha * dotsAlpha).toInt()
            dotPaint.setShadowLayer(2 * d, 0f, 0f, (0x66 * dotsAlpha).toInt() shl 24)
            canvas.drawCircle(x, y, r, dotPaint)
            x += gap
        }
    }

    companion object {
        private const val DOTS_MS = 1200L
    }
}

/** Shown in place of a widget that cannot be displayed (its app was removed or it was unbound). */
@SuppressLint("ViewConstructor")
class WidgetPlaceholderView(context: Context) : TextView(context) {
    init {
        val d = resources.displayMetrics.density
        gravity = Gravity.CENTER
        textSize = 13f
        setTextColor(0xFFFFFFFF.toInt())
        setPadding((12 * d).toInt(), (12 * d).toInt(), (12 * d).toInt(), (12 * d).toInt())
        background = GradientDrawable().apply {
            cornerRadius = 20 * d
            setColor(0x59000000)
            setStroke((1 * d).toInt().coerceAtLeast(1), 0x40FFFFFF)
        }
    }

    /** [label]: the provider's app, if known. [canRestore]: tapping sets the widget up again. */
    fun bind(label: String?, canRestore: Boolean) {
        val res = context.resources
        text = when {
            canRestore -> res.getString(R.string.widget_restore, label ?: res.getString(R.string.widget_generic))
            else -> res.getString(R.string.widget_unavailable)
        }
    }
}
