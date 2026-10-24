package io.github.jacomartins.canyonui.ui.home

import android.graphics.Matrix
import android.view.View

/**
 * Maps points between a root view and a descendant, through scroll offsets and view transforms
 * (scale in edit mode, page scroll). Reuses one matrix; call on the main thread only.
 */
object ViewMap {
    private val inverse = Matrix()

    /** [pt] in [root] coordinates becomes [target]-local coordinates. */
    fun rootToLocal(root: View, target: View, pt: FloatArray) {
        val parent = target.parent as? View ?: return
        if (parent !== root) rootToLocal(root, parent, pt)
        pt[0] += parent.scrollX - target.left
        pt[1] += parent.scrollY - target.top
        val m = target.matrix
        if (!m.isIdentity) {
            m.invert(inverse)
            inverse.mapPoints(pt)
        }
    }

    /** [pt] in [target]-local coordinates becomes [root] coordinates. */
    fun localToRoot(root: View, target: View, pt: FloatArray) {
        var v: View = target
        while (v !== root) {
            val m = v.matrix
            if (!m.isIdentity) m.mapPoints(pt)
            val parent = v.parent as? View ?: return
            pt[0] += v.left - parent.scrollX
            pt[1] += v.top - parent.scrollY
            v = parent
        }
    }

    /** Uniform on-screen scale of [target] relative to [root] (edit mode zoom). */
    fun scaleOf(root: View, target: View): Float {
        var s = 1f
        var v: View = target
        while (v !== root) {
            s *= v.scaleX
            v = v.parent as? View ?: break
        }
        return s
    }
}
