package io.github.jacomartins.canyonui.util

import android.animation.ValueAnimator
import androidx.dynamicanimation.animation.SpringForce

/** Shared motion rules: spring constants and the system "remove animations" switch. */
object Motion {

    /**
     * False when the user turned animations off (developer options or the accessibility
     * "Remove animations" switch). Springs do not follow the animator duration scale on their own,
     * so callers jump straight to the end state instead.
     */
    fun enabled(): Boolean = ValueAnimator.areAnimatorsEnabled()

    /** Page settle: quick, no visible overshoot. */
    const val PAGE_STIFFNESS = 700f
    const val PAGE_DAMPING = 0.9f

    /** Drawer open/close: critically damped so the sheet never overshoots past its edge. */
    const val DRAWER_STIFFNESS = 500f
    const val DRAWER_DAMPING = SpringForce.DAMPING_RATIO_NO_BOUNCY

    const val PRESS_SCALE = 0.9f
    const val PRESS_DURATION_MS = 110L
}
