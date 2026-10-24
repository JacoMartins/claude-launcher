package io.github.jacomartins.canyonui.gesture

import android.accessibilityservice.AccessibilityService
import android.content.Context
import android.content.Intent
import android.provider.Settings
import android.view.accessibility.AccessibilityEvent

/**
 * Lets home screen gestures lock the phone and pull down the notifications. Android only gives
 * ordinary apps these through an accessibility service the user switches on; it reads nothing
 * on screen and reacts to no events, it only performs those two system actions when asked.
 */
class LauncherAccessibilityService : AccessibilityService() {

    override fun onServiceConnected() {
        instance = this
    }

    override fun onUnbind(intent: Intent?): Boolean {
        instance = null
        return super.onUnbind(intent)
    }

    override fun onDestroy() {
        instance = null
        super.onDestroy()
    }

    override fun onAccessibilityEvent(event: AccessibilityEvent?) = Unit

    override fun onInterrupt() = Unit

    companion object {
        private var instance: LauncherAccessibilityService? = null

        val isOn get() = instance != null

        /** Locks the phone; false when the service is off. */
        fun lock(): Boolean = instance?.performGlobalAction(GLOBAL_ACTION_LOCK_SCREEN) ?: false

        /** Switches the service off from inside the app (banking apps may refuse to open while it is on). */
        fun turnOff() {
            instance?.disableSelf()
        }

        /** Pulls down the notifications; false when the service is off. */
        fun notifications(): Boolean = instance?.performGlobalAction(GLOBAL_ACTION_NOTIFICATIONS) ?: false

        /** Android's accessibility settings, where the user switches the service on. */
        fun openSettings(context: Context) {
            try {
                context.startActivity(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK))
            } catch (e: Exception) {
                // No accessibility settings screen: nothing more to offer.
            }
        }
    }
}
