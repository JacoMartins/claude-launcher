package io.github.jacomartins.canyonui.home

import android.content.Context
import android.graphics.Rect
import android.view.View
import android.widget.FrameLayout
import androidx.core.view.ViewCompat
import androidx.core.view.WindowInsetsCompat

/**
 * Root of the home surface. Stage 0: an empty page that tracks the safe area
 * (status bar, navigation/gesture bar, display cutout) so later content can lay out inside it.
 */
class HomePageView(context: Context) : FrameLayout(context) {

    /** Safe-area insets in px, updated on every insets dispatch. */
    val safeInsets = Rect()

    init {
        // The window draws edge-to-edge; children are padded into the safe area here.
        // WindowInsetsCompat keeps this correct on API 29, where the typed insets API is missing.
        ViewCompat.setOnApplyWindowInsetsListener(this) { v, insets -> applyInsets(v, insets) }
    }

    private fun applyInsets(v: View, insets: WindowInsetsCompat): WindowInsetsCompat {
        val bars = insets.getInsets(
            WindowInsetsCompat.Type.systemBars() or WindowInsetsCompat.Type.displayCutout(),
        )
        safeInsets.set(bars.left, bars.top, bars.right, bars.bottom)
        v.setPadding(bars.left, bars.top, bars.right, bars.bottom)
        return insets
    }
}
