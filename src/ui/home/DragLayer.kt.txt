package io.github.jacomartins.canyonui.ui.home

import android.annotation.SuppressLint
import android.content.Context
import android.view.MotionEvent
import android.view.ScaleGestureDetector
import android.view.VelocityTracker
import android.view.ViewConfiguration
import android.widget.FrameLayout
import kotlin.math.abs

/**
 * Root view. Owns the vertical drawer gesture so it works the same over the home screen, the dock
 * and the drawer itself:
 * - drawer closed: a mostly-vertical upward drag opens it;
 * - drawer open: a downward drag closes it, but only when the drawer list is scrolled to the top
 *   and the touch did not start on the fast scroller ([DrawerTouchState]);
 * - drawer moving: a touch catches it and the finger takes over.
 * Horizontal page swipes are claimed by [Workspace] through requestDisallowInterceptTouchEvent.
 */
class DragLayer(context: Context) : FrameLayout(context) {

    /** Written by the drawer UI, read here on the main thread. */
    class DrawerTouchState {
        var contentAtTop = true
        var blockDrag = false
    }

    lateinit var drawer: DrawerController

    /** Drag and drop of home items; takes the gesture over once a long-press has armed it. */
    var drag: DragController? = null

    /** False in "Home screen only" mode: there is no drawer to drag. */
    var drawerEnabled = true
    val drawerTouch = DrawerTouchState()

    private val slop = ViewConfiguration.get(context).scaledTouchSlop
    private val maxFling = ViewConfiguration.get(context).scaledMaximumFlingVelocity.toFloat()
    private var tracker: VelocityTracker? = null
    private var downX = 0f
    private var downY = 0f
    private var lastY = 0f
    private var dragging = false

    /** Where the current gesture went down, in this view's coordinates. */
    val lastDownX get() = armDownX
    val lastDownY get() = armDownY

    private var armDownX = 0f
    private var armDownY = 0f

    /** Two fingers closing on home (pinch in) and opening (pinch out), like One UI's edit mode gesture. */
    var onPinchIn: (() -> Unit)? = null
    var onPinchOut: (() -> Unit)? = null
    private var pinchScale = 1f
    private var pinchDone = false
    private var pinchCancelled = false
    private val pinch = ScaleGestureDetector(context, object : ScaleGestureDetector.SimpleOnScaleGestureListener() {
        override fun onScaleBegin(detector: ScaleGestureDetector): Boolean {
            pinchScale = 1f
            return drawer.isClosed
        }

        override fun onScale(detector: ScaleGestureDetector): Boolean {
            if (pinchDone) return true
            pinchScale *= detector.scaleFactor
            val action = when {
                pinchScale < PINCH_IN -> onPinchIn
                pinchScale > PINCH_OUT -> onPinchOut
                else -> null
            } ?: return true
            pinchDone = true
            action()
            return true
        }
    }).apply { isQuickScaleEnabled = false }

    /**
     * Item drags are handled here rather than in [onInterceptTouchEvent]: scrolling lists in the
     * drawer and folder popup disallow intercepting once they start moving, which would otherwise
     * swallow a drag armed by a long-press on one of their icons.
     */
    override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
        if (ev.actionMasked == MotionEvent.ACTION_DOWN) {
            pinchDone = false
            pinchCancelled = false
        }
        if (drag?.isDragging != true && drawer.isClosed) {
            pinch.onTouchEvent(ev)
            if (pinchDone) {
                // The pinch was used: whatever the fingers started below (a page swipe, a
                // long-press) lets go, and the rest of the gesture goes nowhere.
                if (!pinchCancelled) {
                    pinchCancelled = true
                    val cancel = MotionEvent.obtain(ev)
                    cancel.action = MotionEvent.ACTION_CANCEL
                    super.dispatchTouchEvent(cancel)
                    cancel.recycle()
                    drag?.disarm()
                }
                return true
            }
        }
        val d = drag ?: return super.dispatchTouchEvent(ev)
        if (d.isDragging) {
            when (ev.actionMasked) {
                MotionEvent.ACTION_MOVE -> d.move(ev.x, ev.y)
                MotionEvent.ACTION_UP -> d.drop(ev.x, ev.y)
                MotionEvent.ACTION_CANCEL -> d.cancel()
            }
            return true
        }
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> { armDownX = ev.x; armDownY = ev.y }
            MotionEvent.ACTION_MOVE -> if (d.armed != null && (abs(ev.x - armDownX) > slop || abs(ev.y - armDownY) > slop)) {
                d.begin(ev.x, ev.y)
                // The views under the finger let go of the gesture.
                val cancel = MotionEvent.obtain(ev)
                cancel.action = MotionEvent.ACTION_CANCEL
                super.dispatchTouchEvent(cancel)
                cancel.recycle()
                return true
            }
        }
        val handled = super.dispatchTouchEvent(ev)
        if (ev.actionMasked == MotionEvent.ACTION_UP || ev.actionMasked == MotionEvent.ACTION_CANCEL) d.disarm()
        return handled
    }

    /** Home screen only mode: an upward swipe on home (where the drawer would open) calls this. */
    var onSwipeUp: (() -> Unit)? = null
    private var swipeFired = false

    /** A downward swipe on home (drawer closed) calls this when set; it returns true if used. */
    var onSwipeDown: (() -> Boolean)? = null
    private var downFired = false

    /** A panel over home is open (folder, menu, widget picker...): home's swipes wait until it closes. */
    var homePanelOpen: () -> Boolean = { false }

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        if (drag?.armed != null) return false
        // The panel takes the touches, not the swipes to search, the Apps screen or notifications.
        if (drawer.isClosed && homePanelOpen()) return false
        if (swipeDown(ev)) return true
        if (!drawerEnabled && drawer.isClosed) return swipeUp(ev)
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                downX = ev.x; downY = ev.y; lastY = ev.y
                dragging = false
                track(ev, reset = true)
                if (drawer.isAnimating) {
                    begin(ev)
                    return true
                }
            }
            MotionEvent.ACTION_MOVE -> {
                track(ev)
                val dx = ev.x - downX
                val dy = ev.y - downY
                val vertical = abs(dy) > slop && abs(dy) > abs(dx) * 1.2f
                if (!vertical) return false
                val wantsDrag = when {
                    drawer.isClosed -> dy < 0
                    drawer.isOpen -> dy > 0 && drawerTouch.contentAtTop && !drawerTouch.blockDrag
                    else -> true
                }
                if (wantsDrag) {
                    begin(ev)
                    return true
                }
            }
            MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> recycle()
        }
        return false
    }

    @SuppressLint("ClickableViewAccessibility") // The drawer also opens from a button and accessibility actions.
    override fun onTouchEvent(ev: MotionEvent): Boolean {
        if (!dragging) return super.onTouchEvent(ev)
        track(ev)
        when (ev.actionMasked) {
            MotionEvent.ACTION_MOVE -> {
                drawer.dragBy(ev.y - lastY)
                lastY = ev.y
            }
            MotionEvent.ACTION_UP -> {
                tracker?.computeCurrentVelocity(1000, maxFling)
                drawer.endDrag(tracker?.yVelocity ?: 0f)
                dragging = false
                recycle()
            }
            MotionEvent.ACTION_CANCEL -> {
                drawer.endDrag(0f)
                dragging = false
                recycle()
            }
        }
        return true
    }

    /** Watches for a clearly downward swipe on the closed drawer's home and fires once. */
    private fun swipeDown(ev: MotionEvent): Boolean {
        val callback = onSwipeDown ?: return false
        if (!drawer.isClosed) return false
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                downX = ev.x; downY = ev.y
                downFired = false
            }
            MotionEvent.ACTION_MOVE -> {
                val dx = ev.x - downX
                val dy = ev.y - downY
                if (!downFired && dy > slop * 3 && abs(dy) > abs(dx) * 1.5f) {
                    downFired = true
                    return callback()
                }
            }
        }
        return false
    }

    /** Watches for a clearly upward swipe and takes the gesture once it fires. */
    private fun swipeUp(ev: MotionEvent): Boolean {
        val callback = onSwipeUp ?: return false
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                downX = ev.x; downY = ev.y
                swipeFired = false
            }
            MotionEvent.ACTION_MOVE -> {
                val dx = ev.x - downX
                val dy = ev.y - downY
                if (!swipeFired && dy < -slop * 3 && abs(dy) > abs(dx) * 1.5f) {
                    swipeFired = true
                    callback()
                    return true
                }
            }
        }
        return false
    }

    private fun begin(ev: MotionEvent) {
        dragging = true
        lastY = ev.y
        drawer.startDrag()
    }

    private fun track(ev: MotionEvent, reset: Boolean = false) {
        val t = tracker ?: VelocityTracker.obtain().also { tracker = it }
        if (reset) t.clear()
        t.addMovement(ev)
    }

    private fun recycle() {
        tracker?.recycle()
        tracker = null
    }

    private companion object {
        const val PINCH_IN = 0.8f
        const val PINCH_OUT = 1.25f
    }
}
