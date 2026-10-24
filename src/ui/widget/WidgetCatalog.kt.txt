package io.github.jacomartins.canyonui.ui.widget

import android.appwidget.AppWidgetManager
import android.appwidget.AppWidgetProviderInfo
import android.content.Context
import android.graphics.Bitmap
import android.os.Build
import android.os.UserHandle
import android.os.UserManager
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.asImageBitmap
import androidx.core.graphics.drawable.toBitmap
import io.github.jacomartins.canyonui.data.AppInfo
import java.text.Collator
import kotlin.math.max

/** One widget offered in the picker, with the size it would start at. */
class PickerWidget(
    val info: AppWidgetProviderInfo,
    val label: String,
    val description: String?,
    val spanX: Int,
    val spanY: Int,
    val range: SpanRange,
)

/** The widgets of one app (per profile). */
class PickerGroup(val packageName: String, val user: UserHandle, val label: String, val icon: ImageBitmap?, val widgets: List<PickerWidget>)

/** Lists the home screen widgets installed for every profile, grouped by app. */
object WidgetCatalog {

    fun load(context: Context, apps: List<AppInfo>, sizes: WidgetSizes): List<PickerGroup> {
        val awm = AppWidgetManager.getInstance(context)
        val um = context.getSystemService(UserManager::class.java)
        val pm = context.packageManager
        val infos = um.userProfiles.flatMap { profile ->
            try {
                awm.getInstalledProvidersForProfile(profile)
            } catch (e: RuntimeException) {
                emptyList()
            }
        }.filter { info ->
            info.widgetCategory and AppWidgetProviderInfo.WIDGET_CATEGORY_HOME_SCREEN != 0 &&
                info.widgetFeatures and AppWidgetProviderInfo.WIDGET_FEATURE_HIDE_FROM_PICKER == 0
        }
        val appByPackage = apps.associateBy { it.key.packageName to it.user }
        val collator = Collator.getInstance()
        return infos.groupBy { it.provider.packageName to it.profile }.map { (key, list) ->
            val (pkg, user) = key
            val app = appByPackage[pkg to user]
            val label = app?.label ?: try {
                pm.getApplicationLabel(pm.getApplicationInfo(pkg, 0)).toString()
            } catch (e: Exception) {
                pkg
            }
            val widgets = list.map { info ->
                val (sx, sy) = sizes.defaultSpans(info)
                val description = if (Build.VERSION.SDK_INT >= 31) info.loadDescription(context)?.toString() else null
                PickerWidget(info, info.loadLabel(pm) ?: label, description, sx, sy, sizes.range(info))
            }.sortedWith(compareBy({ it.spanX * it.spanY }, { it.label }))
            PickerGroup(pkg, user, label, app?.image, widgets)
        }.sortedWith { a, b -> collator.compare(a.label, b.label) }
    }

    /** The provider's preview image (or its icon), at most [maxPx] on its long side. */
    fun previewBitmap(context: Context, info: AppWidgetProviderInfo, maxPx: Int): Bitmap? {
        val d = try {
            info.loadPreviewImage(context, 0) ?: info.loadIcon(context, context.resources.displayMetrics.densityDpi)
        } catch (e: RuntimeException) {
            null
        } ?: return null
        val w = d.intrinsicWidth.takeIf { it > 0 } ?: maxPx
        val h = d.intrinsicHeight.takeIf { it > 0 } ?: maxPx
        val s = minOf(1f, maxPx.toFloat() / max(w, h))
        return try {
            d.toBitmap((w * s).toInt().coerceAtLeast(1), (h * s).toInt().coerceAtLeast(1))
        } catch (e: RuntimeException) {
            null
        }
    }

    /**
     * The preview the widget generated for the home screen (Android 15+), or null when it has
     * none or Android does not hand it to this app.
     */
    fun generatedPreview(context: Context, info: AppWidgetProviderInfo): android.widget.RemoteViews? {
        if (Build.VERSION.SDK_INT < 35) return null
        if (info.generatedPreviewCategories and AppWidgetProviderInfo.WIDGET_CATEGORY_HOME_SCREEN == 0) return null
        return try {
            AppWidgetManager.getInstance(context).getWidgetPreview(info.provider, info.profile, AppWidgetProviderInfo.WIDGET_CATEGORY_HOME_SCREEN)
        } catch (e: RuntimeException) {
            null
        }
    }

    /** True when the provider has a real preview image (not just its icon). */
    fun hasPreviewImage(info: AppWidgetProviderInfo) = info.previewImage != 0

    fun previewImage(context: Context, info: AppWidgetProviderInfo, maxPx: Int): ImageBitmap? =
        previewBitmap(context, info, maxPx)?.asImageBitmap()
}
