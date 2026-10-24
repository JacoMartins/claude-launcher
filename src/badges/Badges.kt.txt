package io.github.jacomartins.canyonui.badges

import android.app.Notification
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.provider.Settings
import android.service.notification.NotificationListenerService
import android.service.notification.StatusBarNotification
import androidx.core.app.NotificationManagerCompat
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

/** Notification counts per app (package), for the badges on icons. */
object Badges {
    private val _counts = MutableStateFlow<Map<String, Int>>(emptyMap())
    val counts: StateFlow<Map<String, Int>> = _counts.asStateFlow()

    internal fun publish(counts: Map<String, Int>) {
        _counts.value = counts
    }

    /** True when the user has given Canyon UI notification access. */
    fun hasAccess(context: Context): Boolean = context.packageName in NotificationManagerCompat.getEnabledListenerPackages(context)

    /** Android's notification access screen, where the user allows it. */
    fun openAccessSettings(context: Context) {
        val intent = Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        try {
            context.startActivity(intent)
        } catch (e: Exception) {
            // No such screen on this phone.
        }
    }
}

/**
 * Counts each app's notifications for the badges. It keeps only the counts, in memory: no
 * notification text is read or stored. Ongoing ones (music, calls, downloads), group summaries
 * and apps whose channel hides badges are left out; an app's own count ("3 new messages") is used
 * when it gives one.
 */
class BadgeService : NotificationListenerService() {

    override fun onListenerConnected() = publish()

    override fun onListenerDisconnected() = Badges.publish(emptyMap())

    override fun onNotificationPosted(sbn: StatusBarNotification?) = publish()

    override fun onNotificationRemoved(sbn: StatusBarNotification?) = publish()

    private fun publish() {
        val active = try {
            activeNotifications ?: return
        } catch (e: RuntimeException) {
            return
        }
        val ranking = try {
            currentRanking
        } catch (e: RuntimeException) {
            null
        }
        val r = Ranking()
        val counts = HashMap<String, Int>()
        for (sbn in active) {
            val n = sbn.notification
            if (sbn.isOngoing || !sbn.isClearable) continue
            if (n.flags and Notification.FLAG_GROUP_SUMMARY != 0) continue
            if (ranking != null && ranking.getRanking(sbn.key, r) && !r.canShowBadge()) continue
            counts[sbn.packageName] = (counts[sbn.packageName] ?: 0) + maxOf(1, n.number)
        }
        Badges.publish(counts)
    }

    companion object {
        fun component(context: Context) = ComponentName(context, BadgeService::class.java)
    }
}
