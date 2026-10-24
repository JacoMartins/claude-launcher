package io.github.jacomartins.canyonui.ui.home

import android.annotation.SuppressLint
import android.content.Context
import android.os.Bundle
import android.view.MotionEvent
import android.view.VelocityTracker
import android.view.View
import android.view.ViewConfiguration
import android.view.ViewGroup
import android.view.accessibility.AccessibilityNodeInfo
import androidx.dynamicanimation.animation.FloatPropertyCompat
import androidx.dynamicanimation.animation.SpringAnimation
import androidx.dynamicanimation.animation.SpringForce
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.util.Motion
import kotlin.math.abs
import kotlin.math.roundToInt

/**
 * Horizontally paged home screen. Pages are [CellLayout] children laid out side by side; the
 * finger drives scrollX directly and release settles with a spring that keeps the fling velocity.
 * A touch during the settle catches the page where it is. Vertical gestures are left to the parent.
 */
class Workspace(context: Context) : ViewGroup(context) {

    /** Called with the scroll position in pages (0f = first page, fractional while moving). */
    var onScroll: ((Float) -> Unit)? = null
    var onPageSettled: ((Int) -> Unit)? = null

    var currentPage = 0
        private set

    private val pageCount get() = childCount
    private val isRtl get() = layoutDirection == LAYOUT_DIRECTION_RTL

    private val vc = ViewConfiguration.get(context)
    private val touchSlop = vc.scaledTouchSlop
    private val minFling = vc.scaledMinimumFlingVelocity * 4
    private val maxFling = vc.scaledMaximumFlingVelocity.toFloat()

    private var velocityTracker: VelocityTracker? = null
    private var downX = 0f
    private var downY = 0f
    private var lastX = 0f
    private var dragging = false
    private var dragStartPage = 0

    private val spring = SpringAnimation(this, SCROLL_X).apply {
        spring = SpringForce().setStiffness(Motion.PAGE_STIFFNESS).setDampingRatio(Motion.PAGE_DAMPING)
        setMinimumVisibleChange(1f)
        addEndListener { _, canceled, _, _ -> if (!canceled) onPageSettled?.invoke(currentPage) }
    }

    init {
        isFocusable = true
        importantForAccessibility = IMPORTANT_FOR_ACCESSIBILITY_YES
    }

    fun pageAt(index: Int): CellLayout? = getChildAt(index) as? CellLayout

    /** Visual slot of a page: mirrored in RTL so page 0 is always "first" in reading order. */
    private fun slotOf(page: Int) = if (isRtl) pageCount - 1 - page else page
    private fun pageOfSlot(slot: Int) = if (isRtl) pageCount - 1 - slot else slot
    private fun scrollForPage(page: Int) = slotOf(page) * width
    private val maxScroll get() = ((pageCount - 1).coerceAtLeast(0)) * width

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val w = MeasureSpec.getSize(widthMeasureSpec)
        val h = MeasureSpec.getSize(heightMeasureSpec)
        setMeasuredDimension(w, h)
        val cw = MeasureSpec.makeMeasureSpec(w, MeasureSpec.EXACTLY)
        val ch = MeasureSpec.makeMeasureSpec(h, MeasureSpec.EXACTLY)
        for (i in 0 until childCount) getChildAt(i).measure(cw, ch)
    }

    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        val w = r - l
        for (i in 0 until childCount) {
            val left = slotOf(i) * w
            getChildAt(i).layout(left, 0, left + w, b - t)
        }
        if (!spring.isRunning && !dragging) {
            currentPage = currentPage.coerceIn(0, (pageCount - 1).coerceAtLeast(0))
            scrollTo(scrollForPage(currentPage), 0)
        }
    }

    override fun onScrollChanged(l: Int, t: Int, oldl: Int, oldt: Int) {
        super.onScrollChanged(l, t, oldl, oldt)
        if (width == 0) return
        val slotPos = l.toFloat() / width
        onScroll?.invoke(if (isRtl) (pageCount - 1) - slotPos else slotPos)
    }

    // --- Touch -----------------------------------------------------------------------------

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        if (pageCount <= 1 && ev.actionMasked != MotionEvent.ACTION_DOWN) return false
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                downX = ev.x; downY = ev.y; lastX = ev.x
                trackVelocity(ev, reset = true)
                if (spring.isRunning) {
                    // Catch the page mid-settle and keep following the finger from here.
                    spring.cancel()
                    beginDrag()
                    return true
                }
            }
            MotionEvent.ACTION_MOVE -> {
                trackVelocity(ev)
                val dx = ev.x - downX
                val dy = ev.y - downY
                if (abs(dx) > touchSlop && abs(dx) > abs(dy)) {
                    lastX = ev.x
                    beginDrag()
                    return true
                }
            }
            MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> recycleTracker()
        }
        return false
    }

    @SuppressLint("ClickableViewAccessibility") // Paging is also exposed through accessibility scroll actions.
    override fun onTouchEvent(ev: MotionEvent): Boolean {
        trackVelocity(ev)
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> return true
            MotionEvent.ACTION_MOVE -> {
                if (!dragging) {
                    if (abs(ev.x - downX) > touchSlop) beginDrag() else return true
                }
                var delta = lastX - ev.x
                lastX = ev.x
                val sx = scrollX
                if ((sx <= 0 && delta < 0) || (sx >= maxScroll && delta > 0)) delta *= EDGE_RESISTANCE
                scrollTo((sx + delta).roundToInt(), 0)
            }
            MotionEvent.ACTION_UP -> {
                val vt = velocityTracker
                vt?.computeCurrentVelocity(1000, maxFling)
                endDrag(vt?.xVelocity ?: 0f)
            }
            MotionEvent.ACTION_CANCEL -> endDrag(0f)
        }
        return true
    }

    private fun beginDrag() {
        if (dragging) return
        dragging = true
        dragStartPage = currentPage
        parent?.requestDisallowInterceptTouchEvent(true)
    }

    private fun endDrag(velocityX: Float) {
        if (!dragging) {
            recycleTracker()
            return
        }
        dragging = false
        val w = width.coerceAtLeast(1)
        val slotPos = scrollX.toFloat() / w
        val startSlot = slotOf(dragStartPage)
        val targetSlot = when {
            velocityX < -minFling -> minOf(startSlot + 1, slotPos.toInt() + 1)
            velocityX > minFling -> maxOf(startSlot - 1, kotlin.math.ceil(slotPos).toInt() - 1)
            else -> slotPos.roundToInt()
        }.coerceIn(0, pageCount - 1)
        snapToPage(pageOfSlot(targetSlot), -velocityX)
        recycleTracker()
    }

    /** Animates to [page]. [velocity] is in scroll px/s (positive = towards later slots). */
    fun snapToPage(page: Int, velocity: Float = 0f, animate: Boolean = true) {
        if (pageCount == 0) return
        val target = page.coerceIn(0, pageCount - 1)
        val changed = target != currentPage
        currentPage = target
        val dest = scrollForPage(target).toFloat()
        if (!animate || !Motion.enabled() || width == 0) {
            spring.cancel()
            scrollTo(dest.toInt(), 0)
            onPageSettled?.invoke(target)
        } else {
            spring.setStartVelocity(velocity)
            spring.animateToFinalPosition(dest)
        }
        if (changed) sendAccessibilityEvent(android.view.accessibility.AccessibilityEvent.TYPE_VIEW_SCROLLED)
    }

    private fun trackVelocity(ev: MotionEvent, reset: Boolean = false) {
        val vt = velocityTracker ?: VelocityTracker.obtain().also { velocityTracker = it }
        if (reset) vt.clear()
        vt.addMovement(ev)
    }

    private fun recycleTracker() {
        velocityTracker?.recycle()
        velocityTracker = null
    }

    // --- Accessibility -----------------------------------------------------------------------

    override fun getAccessibilityClassName(): CharSequence = "android.widget.HorizontalScrollView"

    // Left/right are added below and handled in performAccessibilityAction via android.R.id
    // constants, which this lint check does not recognise.
    @SuppressLint("AccessibilityScrollActions")
    override fun onInitializeAccessibilityNodeInfo(info: AccessibilityNodeInfo) {
        super.onInitializeAccessibilityNodeInfo(info)
        info.isScrollable = pageCount > 1
        val next = currentPage < pageCount - 1
        val prev = currentPage > 0
        if (next) {
            info.addAction(AccessibilityNodeInfo.AccessibilityAction.ACTION_SCROLL_FORWARD)
            info.addAction(if (isRtl) AccessibilityNodeInfo.AccessibilityAction.ACTION_SCROLL_LEFT else AccessibilityNodeInfo.AccessibilityAction.ACTION_SCROLL_RIGHT)
        }
        if (prev) {
            info.addAction(AccessibilityNodeInfo.AccessibilityAction.ACTION_SCROLL_BACKWARD)
            info.addAction(if (isRtl) AccessibilityNodeInfo.AccessibilityAction.ACTION_SCROLL_RIGHT else AccessibilityNodeInfo.AccessibilityAction.ACTION_SCROLL_LEFT)
        }
        info.contentDescription = context.getString(R.string.home_page_description, currentPage + 1, pageCount)
    }

    @SuppressLint("AccessibilityScrollActions")
    override fun performAccessibilityAction(action: Int, arguments: Bundle?): Boolean {
        val right = android.R.id.accessibilityActionScrollRight
        val left = android.R.id.accessibilityActionScrollLeft
        val delta = when (action) {
            AccessibilityNodeInfo.ACTION_SCROLL_FORWARD -> 1
            AccessibilityNodeInfo.ACTION_SCROLL_BACKWARD -> -1
            right -> if (isRtl) -1 else 1
            left -> if (isRtl) 1 else -1
            else -> return super.performAccessibilityAction(action, arguments)
        }
        snapToPage(currentPage + delta)
        return true
    }

    override fun shouldDelayChildPressedState() = true

    companion object {
        private const val EDGE_RESISTANCE = 0.3f

        private val SCROLL_X = object : FloatPropertyCompat<Workspace>("scrollX") {
            override fun getValue(view: Workspace) = view.scrollX.toFloat()
            override fun setValue(view: Workspace, value: Float) = view.scrollTo(value.roundToInt(), 0)
        }
    }
}
