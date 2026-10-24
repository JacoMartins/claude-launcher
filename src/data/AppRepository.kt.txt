package io.github.jacomartins.canyonui.data

import android.content.ComponentName
import android.content.Context
import android.content.pm.ApplicationInfo
import android.content.pm.LauncherActivityInfo
import android.content.pm.LauncherApps
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.drawable.AdaptiveIconDrawable
import android.os.Handler
import android.os.Looper
import android.os.Process
import android.os.UserHandle
import android.os.UserManager
import android.util.LruCache
import androidx.compose.runtime.Immutable
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.asImageBitmap
import androidx.core.graphics.createBitmap
import androidx.core.graphics.drawable.toBitmap
import androidx.core.graphics.drawable.toDrawable
import io.github.jacomartins.canyonui.model.AppCategories
import io.github.jacomartins.canyonui.model.AppKey
import io.github.jacomartins.canyonui.model.AppSearch
import kotlinx.coroutines.CompletableDeferred
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

/** A launchable activity with its label and rendered icon. */
@Immutable
class AppInfo(
    val key: AppKey,
    val component: ComponentName,
    val user: UserHandle,
    val label: String,
    val icon: Bitmap,
    val isSystem: Boolean,
    /** A dynamic clock: [icon] is redrawn in place every minute (see [AppRepository.clockTicks]). */
    val isClock: Boolean = false,
    /** Where "Organize apps by category" files it (the user's choice first); null = no category. */
    val category: AppCategories.Category? = null,
    /** The category found automatically, ignoring the user's choice. */
    val autoCategory: AppCategories.Category? = null,
) {
    /** Compose wrapper around [icon]; created once, no copy. */
    val image: ImageBitmap = icon.asImageBitmap()
    val isMainProfile: Boolean get() = user == Process.myUserHandle()
}

/**
 * Every launchable activity in every profile, kept current through [LauncherApps.Callback].
 * Loading and icon rendering run on [Dispatchers.Default]; bursts of package events are merged.
 */
class AppRepository(
    private val context: Context,
    private val scope: CoroutineScope,
    val icons: IconCache,
) {
    private val launcherApps = context.getSystemService(LauncherApps::class.java)
    private val userManager = context.getSystemService(UserManager::class.java)

    private val _apps = MutableStateFlow<List<AppInfo>?>(null)

    /** Sorted A-Z. Null until the first load finishes. */
    val apps: StateFlow<List<AppInfo>?> = _apps.asStateFlow()

    private val reloads = Channel<Unit>(Channel.CONFLATED)

    private val callback = object : LauncherApps.Callback() {
        override fun onPackageRemoved(packageName: String, user: UserHandle) = changed(packageName)
        override fun onPackageAdded(packageName: String, user: UserHandle) = changed(packageName)
        override fun onPackageChanged(packageName: String, user: UserHandle) = changed(packageName)
        override fun onPackagesAvailable(packageNames: Array<out String>, user: UserHandle, replacing: Boolean) =
            packageNames.forEach(::changed)
        override fun onPackagesUnavailable(packageNames: Array<out String>, user: UserHandle, replacing: Boolean) =
            packageNames.forEach(::changed)
    }

    private fun changed(packageName: String) {
        icons.invalidatePackage(packageName)
        // The selected pack was updated, removed or re-enabled: re-read it.
        if (packageName == packPackage) setIconPack(packPackage) else reloads.trySend(Unit)
    }

    private var packPackage: String? = null
    private val packReady = CompletableDeferred<Unit>()
    private val styleReady = CompletableDeferred<Unit>()

    /**
     * Loads [packageName]'s icon pack (null = system icons) off the main thread and re-renders.
     * The first call also releases the initial app load, so icons never flash from system to pack.
     */
    fun setIconPack(packageName: String?) {
        packPackage = packageName
        scope.launch {
            val pack = withContext(Dispatchers.Default) { packageName?.let { IconPack.load(context, it) } }
            if (packPackage != packageName) return@launch // A newer choice won.
            icons.setPack(pack)
            packReady.complete(Unit)
            reloads.trySend(Unit)
        }
    }

    private val _clockTicks = MutableStateFlow(0)

    /** Bumped every minute after clock icons were redrawn; views showing them redraw. */
    val clockTicks: StateFlow<Int> = _clockTicks.asStateFlow()

    /** Called every minute on the main thread. */
    fun timeTick() {
        if (icons.tickClocks()) _clockTicks.value++
    }

    /** New icon shape, theming or per-app choices: re-render every icon. */
    fun setIconStyle(style: IconStyle, overrides: Map<String, IconOverride>) {
        styleReady.complete(Unit)
        if (style == icons.style && overrides == lastOverrides) return
        lastOverrides = overrides
        icons.setStyle(style, overrides)
        reloads.trySend(Unit)
    }

    private var lastOverrides: Map<String, IconOverride> = emptyMap()

    /** Light/dark mode or the wallpaper palette changed (themed icons follow them). */
    fun onConfigurationChanged() {
        icons.onConfigurationChanged()
        reloads.trySend(Unit)
    }

    /** Midnight or a clock change: pack calendar icons show the new day. */
    fun dateChanged() {
        icons.invalidateCalendars()
        reloads.trySend(Unit)
    }

    fun start() {
        launcherApps.registerCallback(callback, Handler(Looper.getMainLooper()))
        scope.launch {
            // Both the pack and the icon style are known before the first render: no icon flash.
            packReady.await()
            styleReady.await()
            for (unused in reloads) {
                _apps.value = withContext(Dispatchers.Default) { load() }
                delay(RELOAD_COALESCE_MS)
            }
        }
    }

    /** Labels depend on the locale. */
    fun reload() {
        reloads.trySend(Unit)
    }

    private fun load(): List<AppInfo> {
        val pm = context.packageManager
        val out = ArrayList<AppInfo>()
        for (profile in launcherApps.profiles) {
            val serial = userManager.getSerialNumberForUser(profile)
            // A locked or paused profile (work profile, Secure Folder) can refuse the query.
            val activities = try {
                launcherApps.getActivityList(null, profile)
            } catch (e: SecurityException) {
                emptyList()
            }
            for (info in activities) {
                val cn = info.componentName
                val key = AppKey(cn.packageName, cn.className, serial)
                val flags = info.applicationInfo.flags
                out += AppInfo(
                    key = key,
                    component = cn,
                    user = profile,
                    label = icons.labelOverride(key) ?: info.label.toString().trim().ifEmpty { cn.packageName },
                    icon = icons.get(info, key, pm),
                    isClock = icons.isClock(key),
                    autoCategory = AppCategories.categorize(cn.packageName, info.applicationInfo.category),
                    category = icons.categoryOverride(key) ?: AppCategories.categorize(cn.packageName, info.applicationInfo.category),
                    isSystem = flags and ApplicationInfo.FLAG_SYSTEM != 0 &&
                        flags and ApplicationInfo.FLAG_UPDATED_SYSTEM_APP == 0,
                )
            }
        }
        return AppSearch.sortByLabel(out) { it.label }
    }

    companion object {
        private const val RELOAD_COALESCE_MS = 250L
    }
}
