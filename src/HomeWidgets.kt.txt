package io.github.jacomartins.canyonui

import android.app.Activity
import android.appwidget.AppWidgetManager
import android.appwidget.AppWidgetProviderInfo
import android.content.ActivityNotFoundException
import android.content.ComponentName
import android.content.Intent
import android.os.Build
import android.os.Bundle
import android.os.UserHandle
import android.os.UserManager
import android.util.Log
import android.view.View
import androidx.core.os.BundleCompat
import io.github.jacomartins.canyonui.data.NewWidget
import io.github.jacomartins.canyonui.data.StackEntry
import io.github.jacomartins.canyonui.data.WidgetEntry
import io.github.jacomartins.canyonui.data.HomeEntry
import io.github.jacomartins.canyonui.model.Container
import io.github.jacomartins.canyonui.model.DropAction
import io.github.jacomartins.canyonui.ui.widget.LauncherWidgetHost
import io.github.jacomartins.canyonui.ui.widget.LauncherWidgetHostView
import io.github.jacomartins.canyonui.ui.widget.SpanRange
import io.github.jacomartins.canyonui.ui.widget.WidgetPlaceholderView
import io.github.jacomartins.canyonui.ui.widget.WidgetSizes

/**
 * The home screen's widgets: the host, one view per widget (reused across rebinds), and the
 * add flow — allocate an id, ask the user for permission to bind if needed, run the widget's
 * setup screen if it has one, then hand the result to [onAdded]. A cancelled or failed add
 * releases the id again.
 */
class HomeWidgets(
    private val activity: Activity,
    private val onProvidersChanged: () -> Unit,
    private val onAdded: (PendingAdd) -> Unit,
    private val onFailed: () -> Unit,
) {
    val manager: AppWidgetManager = AppWidgetManager.getInstance(activity)
    val host = LauncherWidgetHost(activity) { onProvidersChanged() }
    private val users = activity.getSystemService(UserManager::class.java)
    private val views = HashMap<Long, View>()

    /** A widget being added (or set up again in place of [restoreItemId]). */
    class PendingAdd(
        val appWidgetId: Int,
        val provider: ComponentName,
        val user: UserHandle,
        val spanX: Int,
        val spanY: Int,
        val action: DropAction,
        val restoreItemId: Long,
    ) {
        fun toNewWidget(serial: Long) = NewWidget(appWidgetId, provider, serial, spanX, spanY)
    }

    private var pending: PendingAdd? = null
    val isAdding get() = pending != null

    fun serialOf(user: UserHandle) = users.getSerialNumberForUser(user)
    fun userOf(serial: Long): UserHandle? = users.getUserForSerialNumber(serial)

    fun info(appWidgetId: Int): AppWidgetProviderInfo? = try {
        manager.getAppWidgetInfo(appWidgetId)
    } catch (e: RuntimeException) {
        null
    }

    /** The installed provider for [provider] in the profile [serial], if any. */
    fun installedProvider(provider: ComponentName, serial: Long): AppWidgetProviderInfo? {
        val user = userOf(serial) ?: return null
        return try {
            manager.getInstalledProvidersForProfile(user).firstOrNull { it.provider == provider }
        } catch (e: RuntimeException) {
            null
        }
    }

    /**
     * True when the widget's app is no longer installed, so the widget should go. Not decided
     * while its profile is paused or locked (its apps are then only hidden, not removed).
     */
    fun appRemoved(entry: WidgetEntry): Boolean {
        if (entry.appWidgetId >= 0 && info(entry.appWidgetId) != null) return false
        val user = userOf(entry.userSerial) ?: return true // The profile itself was removed.
        if (users.isQuietModeEnabled(user) || !users.isUserUnlocked(user)) return false
        return try {
            activity.getSystemService(android.content.pm.LauncherApps::class.java).getApplicationInfo(entry.provider.packageName, 0, user)
            false
        } catch (e: android.content.pm.PackageManager.NameNotFoundException) {
            true
        } catch (e: RuntimeException) {
            false
        }
    }

    // ------------------------------------------------------------------------------- views

    /**
     * The view for [entry]: the provider's widget, or a placeholder when it cannot be shown.
     * [setup] wires listeners on a newly made view.
     */
    fun viewFor(entry: WidgetEntry, setup: (View) -> Unit): View {
        val existing = views[entry.id]
        if (existing != null && existing.getTag(R.id.tag_widget_id) == entry.appWidgetId) return existing
        (existing?.parent as? android.view.ViewGroup)?.removeView(existing)
        val info = if (entry.appWidgetId >= 0) info(entry.appWidgetId) else null
        val v: View = info?.let { host.createViewSafely(activity, entry.appWidgetId, it) } ?: WidgetPlaceholderView(activity).apply {
            val restorable = installedProvider(entry.provider, entry.userSerial) != null
            bind(appLabel(entry.provider.packageName), restorable)
        }
        // The launcher keeps its own gap around widgets (the cell inset), so the platform's extra
        // padding for older widgets would only shrink them further.
        if (v is LauncherWidgetHostView) v.setPadding(0, 0, 0, 0)
        v.setTag(R.id.tag_widget_id, entry.appWidgetId)
        v.setTag(R.id.tag_item_id, entry.id)
        v.contentDescription = info?.loadLabel(activity.packageManager)
        setup(v)
        views[entry.id] = v
        return v
    }

    /**
     * Forgets views of widgets no longer on home. With [recheck], also placeholders (their app
     * may be back) and widgets the system no longer knows (their app was removed), so the next
     * bind makes them again as what they now are.
     */
    fun prune(keep: Set<Long>, recheck: Boolean = false) {
        val it = views.entries.iterator()
        while (it.hasNext()) {
            val (id, v) = it.next()
            val stale = recheck && (v is WidgetPlaceholderView || info(v.getTag(R.id.tag_widget_id) as? Int ?: -1) == null)
            if (id !in keep || stale) {
                (v.parent as? android.view.ViewGroup)?.removeView(v)
                it.remove()
            }
        }
    }

    fun forEachView(block: (View) -> Unit) = views.values.forEach(block)

    fun setEditMode(on: Boolean) = views.values.forEach { (it as? LauncherWidgetHostView)?.blockContent = on }

    private fun appLabel(pkg: String): String? = try {
        val pm = activity.packageManager
        pm.getApplicationLabel(pm.getApplicationInfo(pkg, 0)).toString()
    } catch (e: Exception) {
        null
    }

    /** The size range of a widget or a stack (what every widget in it allows). */
    fun range(entry: HomeEntry, sizes: WidgetSizes): SpanRange? = when (entry) {
        is WidgetEntry -> info(entry.appWidgetId)?.let { sizes.range(it) }
        is StackEntry -> entry.widgets.mapNotNull { w -> info(w.appWidgetId)?.let { sizes.range(it) } }
            .takeIf { it.size == entry.widgets.size }?.reduce(SpanRange::intersect)
        else -> null
    }?.takeIf { entry.placed.container == Container.DESKTOP }

    // ------------------------------------------------------------------------------- adding

    fun startAdd(info: AppWidgetProviderInfo, spanX: Int, spanY: Int, action: DropAction, options: Bundle, restoreItemId: Long = 0) {
        if (pending != null) return
        val id = try {
            host.allocateAppWidgetId()
        } catch (e: RuntimeException) {
            Log.w(TAG, "allocateAppWidgetId failed", e)
            onFailed()
            return
        }
        pending = PendingAdd(id, info.provider, info.profile, spanX, spanY, action, restoreItemId)
        val bound = try {
            manager.bindAppWidgetIdIfAllowed(id, info.profile, info.provider, options)
        } catch (e: RuntimeException) {
            false
        }
        if (bound) {
            configureOrFinish()
            return
        }
        // Ask the user once per app ("Allow Home to create widgets and access their data?").
        val intent = Intent(AppWidgetManager.ACTION_APPWIDGET_BIND)
            .putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, id)
            .putExtra(AppWidgetManager.EXTRA_APPWIDGET_PROVIDER, info.provider)
            .putExtra(AppWidgetManager.EXTRA_APPWIDGET_PROVIDER_PROFILE, info.profile)
            .putExtra(AppWidgetManager.EXTRA_APPWIDGET_OPTIONS, options)
        try {
            @Suppress("DEPRECATION")
            activity.startActivityForResult(intent, REQ_BIND)
        } catch (e: ActivityNotFoundException) {
            fail()
        }
    }

    private fun configureOrFinish() {
        val p = pending ?: return
        val info = info(p.appWidgetId)
        if (info == null) {
            fail()
            return
        }
        val optional = Build.VERSION.SDK_INT >= 31 && info.widgetFeatures and AppWidgetProviderInfo.WIDGET_FEATURE_CONFIGURATION_OPTIONAL != 0
        if (info.configure != null && !optional) {
            try {
                host.startAppWidgetConfigureActivityForResult(activity, p.appWidgetId, 0, REQ_CONFIGURE, null)
                return
            } catch (e: RuntimeException) {
                // ActivityNotFoundException or SecurityException: the setup screen cannot open.
                Log.w(TAG, "configure failed for ${info.provider}", e)
                fail()
                return
            }
        }
        finish()
    }

    private fun finish() {
        val p = pending ?: return
        pending = null
        onAdded(p)
    }

    private fun fail() {
        pending?.let { host.deleteIdSafely(it.appWidgetId) }
        pending = null
        onFailed()
    }

    /** Opens a widget's settings again (widgets that declare they can be reconfigured). */
    fun reconfigure(appWidgetId: Int): Boolean {
        val info = info(appWidgetId) ?: return false
        if (!canReconfigure(info)) return false
        return try {
            host.startAppWidgetConfigureActivityForResult(activity, appWidgetId, 0, REQ_RECONFIGURE, null)
            true
        } catch (e: RuntimeException) {
            false
        }
    }

    fun canReconfigure(appWidgetId: Int) = info(appWidgetId)?.let(::canReconfigure) == true

    private fun canReconfigure(info: AppWidgetProviderInfo) = info.configure != null &&
        info.widgetFeatures and AppWidgetProviderInfo.WIDGET_FEATURE_RECONFIGURABLE != 0

    /** Returns true if the result belonged to the widget flow. */
    fun onActivityResult(requestCode: Int, resultCode: Int): Boolean {
        when (requestCode) {
            REQ_BIND -> if (resultCode == Activity.RESULT_OK) configureOrFinish() else fail()
            REQ_CONFIGURE -> if (resultCode == Activity.RESULT_OK) finish() else fail()
            REQ_RECONFIGURE -> Unit
            else -> return false
        }
        return true
    }

    // ------------------------------------------------------------------------------- state

    /** The launcher can be stopped while a widget's setup screen is open; the add resumes. */
    fun save(out: Bundle) {
        val p = pending ?: return
        out.putBundle(KEY_PENDING, Bundle().apply {
            putInt("id", p.appWidgetId)
            putParcelable("provider", p.provider)
            putParcelable("user", p.user)
            putInt("sx", p.spanX)
            putInt("sy", p.spanY)
            putLong("restore", p.restoreItemId)
            when (val a = p.action) {
                is DropAction.Move -> { putInt("kind", 0); putInt("page", a.pageIndex); putInt("x", a.x); putInt("y", a.y) }
                is DropAction.CreateStack -> { putInt("kind", 1); putLong("target", a.targetId) }
                is DropAction.AddToStack -> { putInt("kind", 2); putLong("target", a.stackId) }
                else -> putInt("kind", -1)
            }
        })
    }

    fun restore(saved: Bundle?) {
        val b = saved?.getBundle(KEY_PENDING) ?: return
        val provider = BundleCompat.getParcelable(b, "provider", ComponentName::class.java) ?: return
        val user = BundleCompat.getParcelable(b, "user", UserHandle::class.java) ?: return
        val action = when (b.getInt("kind")) {
            0 -> DropAction.Move(Container.DESKTOP, b.getInt("page"), b.getInt("x"), b.getInt("y"))
            1 -> DropAction.CreateStack(b.getLong("target"))
            2 -> DropAction.AddToStack(b.getLong("target"))
            else -> DropAction.Reject
        }
        pending = PendingAdd(b.getInt("id"), provider, user, b.getInt("sx"), b.getInt("sy"), action, b.getLong("restore"))
    }

    /** Releases host ids that no layout uses (left by removed widgets or interrupted adds). */
    fun releaseUnused(inUse: Set<Int>) {
        if (pending != null) return
        val ids = try {
            host.appWidgetIds
        } catch (e: RuntimeException) {
            return
        }
        for (id in ids) if (id !in inUse) host.deleteIdSafely(id)
    }

    companion object {
        private const val TAG = "HomeWidgets"
        private const val KEY_PENDING = "pending_widget"
        const val REQ_BIND = 4101
        const val REQ_CONFIGURE = 4102
        const val REQ_RECONFIGURE = 4103
    }
}
