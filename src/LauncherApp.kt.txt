package io.github.jacomartins.canyonui

import android.app.Application
import android.content.res.Configuration
import io.github.jacomartins.canyonui.data.AppRepository
import io.github.jacomartins.canyonui.data.AppUsage
import io.github.jacomartins.canyonui.data.FolderEntry
import kotlinx.coroutines.flow.filterNotNull
import kotlinx.coroutines.flow.debounce
import io.github.jacomartins.canyonui.model.AppCategories
import io.github.jacomartins.canyonui.data.DefaultApps
import io.github.jacomartins.canyonui.data.HomeRepository
import io.github.jacomartins.canyonui.data.IconCache
import io.github.jacomartins.canyonui.data.LauncherDatabase
import io.github.jacomartins.canyonui.data.LayoutMode
import io.github.jacomartins.canyonui.data.SettingsStore
import io.github.jacomartins.canyonui.data.titleRes
import kotlinx.coroutines.MainScope
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.launch
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import androidx.core.content.ContextCompat

/** Process-wide objects, created once (manual dependency injection). */
class LauncherApp : Application() {

    val scope = MainScope()

    lateinit var settings: SettingsStore
        private set
    lateinit var apps: AppRepository
        private set
    lateinit var home: HomeRepository
        private set
    lateinit var usage: AppUsage
        private set
    /** The layout databases, one per layout mode (for backups). */
    lateinit var dbs: Map<LayoutMode, LauncherDatabase>
        private set

    override fun onCreate() {
        super.onCreate()
        dbs = LayoutMode.entries.associateWith { LauncherDatabase.create(this, it) }
        settings = SettingsStore(this)
        usage = AppUsage(this)
        apps = AppRepository(this, scope, IconCache(this))
        home = HomeRepository(dbs, settings, apps, DefaultApps(this), scope, ::categoryTitle)
        apps.start()
        home.start()
        // Category folders: keep their apps in order as apps come and go (only when something
        // actually moves). The daily re-check runs when home comes back to the front.
        scope.launch {
            home.state.filterNotNull()
                .map { st -> st.entries.filterIsInstance<FolderEntry>().filter { it.isCategory }.map { f -> f.id to f.apps.map { it.id } } }
                .distinctUntilChanged()
                .debounce(1_000)
                .collect { home.arrangeCategoryFolders(usage) }
        }
        scope.launch {
            settings.settings.map { it.renderMode }.distinctUntilChanged().collect {
                io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.setMode(this@LauncherApp, it)
            }
        }
        scope.launch {
            settings.settings.map { it.iconPack }.distinctUntilChanged().collect { apps.setIconPack(it) }
        }
        scope.launch {
            settings.settings.map { it.iconStyle to it.iconOverrides }.distinctUntilChanged().collect { (style, overrides) ->
                apps.setIconStyle(style, overrides)
            }
        }
        // Dynamic clock icons move once a minute.
        ContextCompat.registerReceiver(
            this,
            object : BroadcastReceiver() {
                override fun onReceive(context: Context, intent: Intent) = apps.timeTick()
            },
            IntentFilter(Intent.ACTION_TIME_TICK),
            ContextCompat.RECEIVER_NOT_EXPORTED,
        )
        val dateFilter = IntentFilter().apply {
            addAction(Intent.ACTION_DATE_CHANGED)
            addAction(Intent.ACTION_TIME_CHANGED)
            addAction(Intent.ACTION_TIMEZONE_CHANGED)
        }
        ContextCompat.registerReceiver(
            this,
            object : BroadcastReceiver() {
                override fun onReceive(context: Context, intent: Intent) = apps.dateChanged()
            },
            dateFilter,
            ContextCompat.RECEIVER_NOT_EXPORTED,
        )
    }

    private fun categoryTitle(c: AppCategories.Category): String = getString(c.titleRes())

    override fun onConfigurationChanged(newConfig: Configuration) {
        super.onConfigurationChanged(newConfig)
        apps.onConfigurationChanged() // Labels follow the locale; themed icons the palette.
    }
}
