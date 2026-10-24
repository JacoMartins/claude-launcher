package io.github.jacomartins.canyonui.ui.home

import android.view.View
import androidx.dynamicanimation.animation.DynamicAnimation
import androidx.dynamicanimation.animation.FloatValueHolder
import androidx.dynamicanimation.animation.SpringAnimation
import androidx.dynamicanimation.animation.SpringForce
import io.github.jacomartins.canyonui.util.Motion

/**
 * Drives the app drawer between closed (0) and open (1). The drawer slides up with the finger and
 * the home screen fades and shrinks slightly behind it; release settles with a spring that keeps
 * the finger's velocity. Only view properties change per frame (translation, alpha, scale), and the
 * home screen is drawn from a hardware layer while it moves.
 */
class DrawerController(
    private val home: View,
    private val drawer: View,
    private val listener: Listener,
) {
    interface Listener {
        fun onDrawerOpened()
        fun onDrawerClosed()
        /** Crossing the halfway point; used for status bar icon contrast. */
        fun onDrawerMostlyOpen(mostlyOpen: Boolean)
    }

    var progress = 0f
        private set

    private var mostlyOpen = false
    private var layersOn = false
    private var target = 0f

    private val holder = FloatValueHolder()
    private val spring = SpringAnimation(holder).apply {
        spring = SpringForce().setStiffness(Motion.DRAWER_STIFFNESS).setDampingRatio(Motion.DRAWER_DAMPING)
        setMinimumVisibleChange(DynamicAnimation.MIN_VISIBLE_CHANGE_SCALE)
        addUpdateListener { _, value, _ -> apply(value) }
        addEndListener { _, canceled, _, _ -> if (!canceled) settled() }
    }

    val isAnimating get() = spring.isRunning
    val isOpen get() = !spring.isRunning && progress >= 1f
    val isClosed get() = !spring.isRunning && progress <= 0f

    private val range: Float get() = (drawer.height.takeIf { it > 0 } ?: home.height).toFloat().coerceAtLeast(1f)

    fun startDrag() {
        spring.cancel()
        setLayers(true)
        drawer.visibility = View.VISIBLE
    }

    /** [dy] in px, positive = finger moving down (closing). */
    fun dragBy(dy: Float) {
        apply((progress - dy / range).coerceIn(0f, 1f))
    }

    /** [velocityY] in px/s, positive = downward. Opens or closes depending on speed and position. */
    fun endDrag(velocityY: Float) {
        val v = -velocityY / range
        val open = when {
            v > FLING_THRESHOLD -> true
            v < -FLING_THRESHOLD -> false
            else -> progress > 0.4f
        }
        animateTo(if (open) 1f else 0f, v)
    }

    fun open(animate: Boolean = true) = animateTo(1f, 0f, animate)
    fun close(animate: Boolean = true) = animateTo(0f, 0f, animate)

    private fun animateTo(to: Float, velocity: Float, animate: Boolean = true) {
        target = to
        if (!animate || !Motion.enabled()) {
            spring.cancel()
            apply(to)
            settled()
            return
        }
        setLayers(true)
        drawer.visibility = View.VISIBLE
        holder.value = progress
        spring.setStartValue(progress)
        spring.setStartVelocity(velocity)
        spring.animateToFinalPosition(to)
    }

    /** Predictive back: shrinks the open drawer a little while the back gesture is in progress. */
    fun previewBack(backProgress: Float) {
        val p = 1f - 0.12f * backProgress
        drawer.translationY = (1f - p) * range
        onOffset?.invoke(drawer.translationY)
    }

    /** Called with the drawer's vertical offset whenever it moves (for its glass background). */
    var onOffset: ((Float) -> Unit)? = null

    fun cancelBackPreview() = apply(progress)

    private fun apply(p: Float) {
        progress = p
        drawer.translationY = (1f - p) * range
        onOffset?.invoke(drawer.translationY)
        home.alpha = 1f - p
        val scale = 1f - HOME_SCALE_DELTA * p
        home.scaleX = scale
        home.scaleY = scale
        val mostly = p > 0.5f
        if (mostly != mostlyOpen) {
            mostlyOpen = mostly
            listener.onDrawerMostlyOpen(mostly)
        }
    }

    private fun settled() {
        setLayers(false)
        if (progress <= 0f) {
            home.visibility = View.VISIBLE
            drawer.visibility = View.INVISIBLE
            listener.onDrawerClosed()
        } else if (progress >= 1f) {
            home.visibility = View.INVISIBLE // Nothing to draw under an opaque drawer.
            listener.onDrawerOpened()
        }
    }

    private fun setLayers(on: Boolean) {
        if (on) home.visibility = View.VISIBLE
        if (on == layersOn) return
        layersOn = on
        home.setLayerType(if (on) View.LAYER_TYPE_HARDWARE else View.LAYER_TYPE_NONE, null)
        // Home is one picture while it moves: its blurs must not redraw it on every frame.
        io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.hold(on)
    }

    companion object {
        private const val HOME_SCALE_DELTA = 0.06f
        /** In drawer heights per second. */
        private const val FLING_THRESHOLD = 0.6f
    }
}
