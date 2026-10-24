package io.github.jacomartins.canyonui.data

import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.provider.AlarmClock
import android.provider.MediaStore
import android.provider.Settings
import androidx.core.net.toUri
import io.github.jacomartins.canyonui.model.AppKey

/**
 * Finds the apps for the first-run layout through standard intents, so it works whatever the
 * phone's maker ships (Samsung, Google or other phone/message/browser apps).
 */
class DefaultApps(private val context: Context) {

    fun dockCandidates(apps: List<AppInfo>): List<AppKey?> = listOf(
        Intent(Intent.ACTION_DIAL),
        Intent.makeMainSelectorActivity(Intent.ACTION_MAIN, Intent.CATEGORY_APP_MESSAGING),
        Intent(Intent.ACTION_VIEW, "https://example.com".toUri()).addCategory(Intent.CATEGORY_BROWSABLE),
        Intent(MediaStore.INTENT_ACTION_STILL_IMAGE_CAMERA),
    ).map { resolve(it, apps) }

    fun pageCandidates(apps: List<AppInfo>): List<AppKey?> = listOf(
        byPackage("com.android.vending", apps),
        resolve(Intent.makeMainSelectorActivity(Intent.ACTION_MAIN, Intent.CATEGORY_APP_GALLERY), apps),
        resolve(Intent.makeMainSelectorActivity(Intent.ACTION_MAIN, Intent.CATEGORY_APP_CALENDAR), apps),
        resolve(Intent(AlarmClock.ACTION_SHOW_ALARMS), apps),
        resolve(Intent(Settings.ACTION_SETTINGS), apps),
    )

    /** The default handler if one is set, else the first handler that has a launcher icon. */
    private fun resolve(intent: Intent, apps: List<AppInfo>): AppKey? {
        val pm = context.packageManager
        val default = resolveActivity(pm, intent)?.activityInfo?.packageName
        if (default != null) byPackage(default, apps)?.let { return it }
        return queryActivities(pm, intent)
            .asSequence()
            .mapNotNull { byPackage(it.activityInfo.packageName, apps) }
            .firstOrNull()
    }

    private fun byPackage(pkg: String, apps: List<AppInfo>): AppKey? =
        apps.firstOrNull { it.key.packageName == pkg && it.isMainProfile }?.key

    private fun resolveActivity(pm: PackageManager, intent: Intent) =
        if (android.os.Build.VERSION.SDK_INT >= 33) {
            pm.resolveActivity(intent, PackageManager.ResolveInfoFlags.of(PackageManager.MATCH_DEFAULT_ONLY.toLong()))
        } else {
            @Suppress("DEPRECATION")
            pm.resolveActivity(intent, PackageManager.MATCH_DEFAULT_ONLY)
        }

    private fun queryActivities(pm: PackageManager, intent: Intent) =
        if (android.os.Build.VERSION.SDK_INT >= 33) {
            pm.queryIntentActivities(intent, PackageManager.ResolveInfoFlags.of(PackageManager.MATCH_DEFAULT_ONLY.toLong()))
        } else {
            @Suppress("DEPRECATION")
            pm.queryIntentActivities(intent, PackageManager.MATCH_DEFAULT_ONLY)
        }
}
