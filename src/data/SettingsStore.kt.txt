package io.github.jacomartins.canyonui.data

import android.content.Context
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.booleanPreferencesKey
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.intPreferencesKey
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.core.stringSetPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.map
import org.json.JSONObject

private val Context.dataStore by preferencesDataStore(name = "settings")

/**
 * How see-through surfaces (folders, panels, menus, edit pages) are drawn: plain [TRANSLUCENT]
 * colour, [FROSTED] over a strongly blurred wallpaper, or [GLASS] over a lightly blurred one.
 */
enum class RenderMode(val storageKey: String) {
    TRANSLUCENT("translucent"),
    FROSTED("frosted"),
    GLASS("glass"),
}

/** "Home and Apps screens" (with a drawer) or "Home screen only" (every app on home). */
enum class LayoutMode(val storageKey: String) {
    HOME_AND_DRAWER("home_drawer"),
    HOME_ONLY("home_only"),
}

/**
 * Icon size as a share of the column width, with an upper bound in dp. Medium matches One UI's
 * 4x6 home screen, where an icon fills about 71% of its column.
 */
enum class IconSize(val cellFraction: Float, val maxDp: Int) {
    SMALL(0.62f, 64),
    MEDIUM(0.72f, 72),
    LARGE(0.82f, 80),
}

/** Grid, dock and icon size of one layout mode. Each mode keeps its own. */
data class GridSpec(
    val columns: Int = 4,
    val rows: Int = 6,
    val dockSlots: Int = 4,
    val iconSize: IconSize = IconSize.MEDIUM,
    /** Labels under home screen icons. The Apps screen always shows them. */
    val showHomeLabels: Boolean = false,
    /** Labels under dock icons, set separately from the home screen. */
    val showDockLabels: Boolean = false,
)

data class LauncherSettings(
    val layoutMode: LayoutMode = LayoutMode.HOME_AND_DRAWER,
    val grids: Map<LayoutMode, GridSpec> = LayoutMode.entries.associateWith { GridSpec() },
    val hiddenByMode: Map<LayoutMode, Set<String>> = LayoutMode.entries.associateWith { emptySet() },
    val firstRunDone: Boolean = false,
    val drawerHintSeen: Boolean = false,
    /** Package of the chosen icon pack; null = system icons. Shared by both layout modes. */
    val iconPack: String? = null,
    /** Lets any widget be resized to any size, ignoring the sizes its app declares. */
    val resizeAnyWidget: Boolean = false,
    /**
     * Tell one-row (and one-column) widgets the whole cell's size instead of the pill's
     * thickness, for widgets that pick small text from the pill's size.
     */
    val fullCellWidgetSizes: Boolean = true,
    /** OneDepth's translucent ring along widget edges. */
    val widgetBorder: Boolean = true,
    /** Interface colours from the phone's palette instead of Canyon UI's own. */
    val phonePalette: Boolean = false,
    /** Notification badges on icons: "off", "dots" or "numbers". */
    val badges: String = "numbers",
    /** Colour of folders that have none of their own (ARGB); 0 = the theme's, light or dark. */
    val folderColor: Int = 0,
    /** Taps on empty home space that lock the phone: 0 (off), 2 or 3. */
    val lockTaps: Int = 0,
    /** A downward swipe on home pulls down the notifications. */
    val swipeDownNotifications: Boolean = false,
    /** How see-through surfaces are drawn; glass by default. */
    val renderMode: RenderMode = RenderMode.GLASS,
    /** Category mode just started and the home page has room for a Google search bar. */
    val pendingSearchWidget: Boolean = false,
    /**
     * Home screen only mode: what a swipe up on home opens, as "action|package/class" (a search
     * app found by its search action, or any app by MAIN); null = nothing.
     */
    val swipeUpTarget: String? = null,
    /** What the Wallpaper button opens (like [swipeUpTarget]); null = Canyon UI's own chooser. */
    val wallpaperTarget: String? = null,
    /** Widget providers (flattened component names) drawn without the ring, chosen per widget. */
    val noOutlineWidgets: Set<String> = emptySet(),
    /** Widget stacks (ids) that turn to their next widget on their own. */
    val slideshowStacks: Set<String> = emptySet(),
    /** Home screen only mode: apps start in category folders; new apps join theirs. */
    val autoCategories: Boolean = false,
    val iconStyle: IconStyle = IconStyle(),
    /** Per-app label and icon choices, by [io.github.jacomartins.canyonui.model.AppKey] string. */
    val iconOverrides: Map<String, IconOverride> = emptyMap(),
) {
    /** Grid of the active mode. */
    val grid: GridSpec get() = grids.getValue(layoutMode)
    val columns get() = grid.columns
    val rows get() = grid.rows
    val dockSlots get() = grid.dockSlots
    val drawerEnabled get() = layoutMode == LayoutMode.HOME_AND_DRAWER

    /** Hidden apps of the active mode. */
    val hiddenApps: Set<String> get() = hiddenByMode.getValue(layoutMode)
}

/**
 * User settings in DataStore. Layout-specific values (grid, dock) are stored per [LayoutMode]
 * under the mode's prefix, so switching modes keeps each mode's own setup.
 */
class SettingsStore(private val context: Context) {

    private object Keys {
        val LAYOUT_MODE = stringPreferencesKey("layout_mode")
        /** Stage 1 kept one hidden list for everything; it seeds both modes' lists. */
        val LEGACY_HIDDEN = stringSetPreferencesKey("hidden_apps")
        val FIRST_RUN_DONE = booleanPreferencesKey("first_run_done")
        val DRAWER_HINT_SEEN = booleanPreferencesKey("drawer_hint_seen")
        val ICON_PACK = stringPreferencesKey("icon_pack")
        val RESIZE_ANY_WIDGET = booleanPreferencesKey("resize_any_widget")
        val WIDGET_BORDER = booleanPreferencesKey("widget_border")
        val FULL_CELL_WIDGET_SIZES = booleanPreferencesKey("full_cell_widget_sizes")
        val PHONE_PALETTE = booleanPreferencesKey("phone_palette")
        val RENDER_MODE = stringPreferencesKey("render_mode")
        val LOCK_TAPS = intPreferencesKey("lock_taps")
        val BADGES = stringPreferencesKey("badges")
        val FOLDER_COLOR = intPreferencesKey("folder_color")
        val SWIPE_DOWN_NOTIFICATIONS = booleanPreferencesKey("swipe_down_notifications")
        val PENDING_SEARCH_WIDGET = booleanPreferencesKey("pending_search_widget")
        val SWIPE_UP_TARGET = stringPreferencesKey("swipe_up_target")
        val WALLPAPER_TARGET = stringPreferencesKey("wallpaper_target")
        val NO_OUTLINE_WIDGETS = stringSetPreferencesKey("no_outline_widgets")
        val SLIDESHOW_STACKS = stringSetPreferencesKey("slideshow_stacks")
        val AUTO_CATEGORIES = booleanPreferencesKey("auto_categories")
        val ICON_SHAPE = stringPreferencesKey("icon_shape")
        val THEMED_ICONS = booleanPreferencesKey("themed_icons")
        val PACK_STYLE = booleanPreferencesKey("pack_style")
        val ICON_OVERRIDES = stringPreferencesKey("icon_overrides")

        fun columns(m: LayoutMode) = intPreferencesKey("${m.storageKey}.grid_columns")
        fun rows(m: LayoutMode) = intPreferencesKey("${m.storageKey}.grid_rows")
        fun dock(m: LayoutMode) = intPreferencesKey("${m.storageKey}.dock_slots")
        fun iconSize(m: LayoutMode) = stringPreferencesKey("${m.storageKey}.icon_size")
        fun showHomeLabels(m: LayoutMode) = booleanPreferencesKey("${m.storageKey}.show_labels")
        fun showDockLabels(m: LayoutMode) = booleanPreferencesKey("${m.storageKey}.show_dock_labels")
        fun hidden(m: LayoutMode) = stringSetPreferencesKey("${m.storageKey}.hidden_apps")
    }

    private fun grid(p: Preferences, m: LayoutMode): GridSpec {
        val d = GridSpec()
        return GridSpec(
            columns = (p[Keys.columns(m)] ?: d.columns).coerceIn(MIN_COLUMNS, MAX_COLUMNS),
            rows = (p[Keys.rows(m)] ?: d.rows).coerceIn(MIN_ROWS, MAX_ROWS),
            dockSlots = (p[Keys.dock(m)] ?: d.dockSlots).coerceIn(4, 5),
            iconSize = p[Keys.iconSize(m)]?.let { v -> IconSize.entries.firstOrNull { it.name == v } } ?: d.iconSize,
            showHomeLabels = p[Keys.showHomeLabels(m)] ?: d.showHomeLabels,
            showDockLabels = p[Keys.showDockLabels(m)] ?: d.showDockLabels,
        )
    }

    val settings: Flow<LauncherSettings> = context.dataStore.data.map { p ->
        val d = LauncherSettings()
        LauncherSettings(
            layoutMode = p[Keys.LAYOUT_MODE]?.let { v -> LayoutMode.entries.firstOrNull { it.name == v } } ?: d.layoutMode,
            grids = LayoutMode.entries.associateWith { grid(p, it) },
            hiddenByMode = LayoutMode.entries.associateWith { p[Keys.hidden(it)] ?: p[Keys.LEGACY_HIDDEN] ?: emptySet() },
            firstRunDone = p[Keys.FIRST_RUN_DONE] ?: false,
            drawerHintSeen = p[Keys.DRAWER_HINT_SEEN] ?: false,
            iconPack = p[Keys.ICON_PACK],
            resizeAnyWidget = p[Keys.RESIZE_ANY_WIDGET] ?: false,
            widgetBorder = p[Keys.WIDGET_BORDER] ?: true,
            fullCellWidgetSizes = p[Keys.FULL_CELL_WIDGET_SIZES] ?: true,
            phonePalette = p[Keys.PHONE_PALETTE] ?: false,
            lockTaps = p[Keys.LOCK_TAPS] ?: 0,
            badges = p[Keys.BADGES] ?: "numbers",
            folderColor = p[Keys.FOLDER_COLOR] ?: 0,
            swipeDownNotifications = p[Keys.SWIPE_DOWN_NOTIFICATIONS] ?: false,
            renderMode = RenderMode.entries.firstOrNull { it.storageKey == p[Keys.RENDER_MODE] } ?: RenderMode.GLASS,
            pendingSearchWidget = p[Keys.PENDING_SEARCH_WIDGET] ?: false,
            swipeUpTarget = p[Keys.SWIPE_UP_TARGET],
            wallpaperTarget = p[Keys.WALLPAPER_TARGET],
            noOutlineWidgets = p[Keys.NO_OUTLINE_WIDGETS] ?: emptySet(),
            slideshowStacks = p[Keys.SLIDESHOW_STACKS] ?: emptySet(),
            autoCategories = p[Keys.AUTO_CATEGORIES] ?: false,
            iconStyle = IconStyle(
                shape = IconShape.fromKey(p[Keys.ICON_SHAPE]),
                themed = p[Keys.THEMED_ICONS] ?: false,
                packStyle = p[Keys.PACK_STYLE] ?: true,
            ),
            iconOverrides = parseOverrides(p[Keys.ICON_OVERRIDES]),
        )
    }.distinctUntilChanged()

    suspend fun current(): LauncherSettings = settings.first()

    suspend fun setLayoutMode(mode: LayoutMode) {
        context.dataStore.edit { it[Keys.LAYOUT_MODE] = mode.name }
    }

    suspend fun setGrid(mode: LayoutMode, columns: Int, rows: Int) {
        context.dataStore.edit {
            it[Keys.columns(mode)] = columns.coerceIn(MIN_COLUMNS, MAX_COLUMNS)
            it[Keys.rows(mode)] = rows.coerceIn(MIN_ROWS, MAX_ROWS)
        }
    }

    suspend fun setIconPack(packageName: String?) {
        context.dataStore.edit { if (packageName == null) it.remove(Keys.ICON_PACK) else it[Keys.ICON_PACK] = packageName }
    }

    suspend fun setIconShape(shape: IconShape) {
        context.dataStore.edit { it[Keys.ICON_SHAPE] = shape.storageKey }
    }

    suspend fun setThemedIcons(on: Boolean) {
        context.dataStore.edit { it[Keys.THEMED_ICONS] = on }
    }

    suspend fun setPackStyle(on: Boolean) {
        context.dataStore.edit { it[Keys.PACK_STYLE] = on }
    }

    /** Sets or (null / empty) clears the label and icon choice for one app. */
    suspend fun setIconOverride(appKey: String, override: IconOverride?) {
        context.dataStore.edit { p ->
            val all = parseOverrides(p[Keys.ICON_OVERRIDES]).toMutableMap()
            if (override == null || override.isEmpty) all.remove(appKey) else all[appKey] = override
            p[Keys.ICON_OVERRIDES] = writeOverrides(all)
        }
    }

    private fun parseOverrides(json: String?): Map<String, IconOverride> {
        if (json.isNullOrEmpty()) return emptyMap()
        return try {
            val o = JSONObject(json)
            o.keys().asSequence().associateWith { k ->
                val e = o.getJSONObject(k)
                IconOverride(
                    label = e.optString("label").ifEmpty { null },
                    packPackage = e.optString("pack").ifEmpty { null },
                    drawable = e.optString("drawable").ifEmpty { null },
                    file = e.optString("file").ifEmpty { null },
                    category = e.optString("category").ifEmpty { null },
                )
            }
        } catch (e: org.json.JSONException) {
            emptyMap()
        }
    }

    private fun writeOverrides(all: Map<String, IconOverride>): String = JSONObject().apply {
        for ((k, v) in all) {
            put(k, JSONObject().apply {
                v.label?.let { put("label", it) }
                v.packPackage?.let { put("pack", it) }
                v.drawable?.let { put("drawable", it) }
                v.file?.let { put("file", it) }
                v.category?.let { put("category", it) }
            })
        }
    }.toString()

    suspend fun setAutoCategories(on: Boolean) {
        context.dataStore.edit { it[Keys.AUTO_CATEGORIES] = on }
    }

    /** Shows or hides the ring for one kind of widget ([provider] as a flattened component). */
    suspend fun setWidgetOutline(provider: String, on: Boolean) {
        context.dataStore.edit {
            val set = it[Keys.NO_OUTLINE_WIDGETS].orEmpty()
            it[Keys.NO_OUTLINE_WIDGETS] = if (on) set - provider else set + provider
        }
    }

    suspend fun setSlideshow(stackId: Long, on: Boolean) {
        context.dataStore.edit {
            val set = it[Keys.SLIDESHOW_STACKS].orEmpty()
            it[Keys.SLIDESHOW_STACKS] = if (on) set + stackId.toString() else set - stackId.toString()
        }
    }

    suspend fun setPendingSearchWidget(on: Boolean) {
        context.dataStore.edit { it[Keys.PENDING_SEARCH_WIDGET] = on }
    }

    suspend fun setSwipeUpTarget(target: String?) {
        context.dataStore.edit { if (target == null) it.remove(Keys.SWIPE_UP_TARGET) else it[Keys.SWIPE_UP_TARGET] = target }
    }


    suspend fun setWallpaperTarget(target: String?) {
        context.dataStore.edit { if (target == null) it.remove(Keys.WALLPAPER_TARGET) else it[Keys.WALLPAPER_TARGET] = target }
    }

    suspend fun setPhonePalette(on: Boolean) {
        context.dataStore.edit { it[Keys.PHONE_PALETTE] = on }
    }

    suspend fun setFolderColor(color: Int) {
        context.dataStore.edit { if (color == 0) it.remove(Keys.FOLDER_COLOR) else it[Keys.FOLDER_COLOR] = color }
    }

    suspend fun setBadges(mode: String) {
        context.dataStore.edit { it[Keys.BADGES] = mode }
    }

    suspend fun setLockTaps(taps: Int) {
        context.dataStore.edit { it[Keys.LOCK_TAPS] = taps }
    }

    suspend fun setSwipeDownNotifications(on: Boolean) {
        context.dataStore.edit { it[Keys.SWIPE_DOWN_NOTIFICATIONS] = on }
    }

    suspend fun setRenderMode(mode: RenderMode) {
        context.dataStore.edit { it[Keys.RENDER_MODE] = mode.storageKey }
    }

    suspend fun setFullCellWidgetSizes(on: Boolean) {
        context.dataStore.edit { it[Keys.FULL_CELL_WIDGET_SIZES] = on }
    }

    suspend fun setWidgetBorder(on: Boolean) {
        context.dataStore.edit { it[Keys.WIDGET_BORDER] = on }
    }

    suspend fun setResizeAnyWidget(on: Boolean) {
        context.dataStore.edit { it[Keys.RESIZE_ANY_WIDGET] = on }
    }

    suspend fun setIconSize(mode: LayoutMode, size: IconSize) {
        context.dataStore.edit { it[Keys.iconSize(mode)] = size.name }
    }

    suspend fun setShowHomeLabels(mode: LayoutMode, show: Boolean) {
        context.dataStore.edit { it[Keys.showHomeLabels(mode)] = show }
    }

    suspend fun setShowDockLabels(mode: LayoutMode, show: Boolean) {
        context.dataStore.edit { it[Keys.showDockLabels(mode)] = show }
    }

    /** Hides or shows an app in the active layout mode only. */
    suspend fun setHidden(key: String, hidden: Boolean) {
        context.dataStore.edit { p ->
            val mode = p[Keys.LAYOUT_MODE]?.let { v -> LayoutMode.entries.firstOrNull { it.name == v } } ?: LayoutMode.HOME_AND_DRAWER
            val set = (p[Keys.hidden(mode)] ?: p[Keys.LEGACY_HIDDEN] ?: emptySet()).toMutableSet()
            if (hidden) set += key else set -= key
            p[Keys.hidden(mode)] = set
        }
    }

    suspend fun setFirstRunDone() {
        context.dataStore.edit { it[Keys.FIRST_RUN_DONE] = true }
    }

    suspend fun setDrawerHintSeen() {
        context.dataStore.edit { it[Keys.DRAWER_HINT_SEEN] = true }
    }

    companion object {
        const val MIN_COLUMNS = 4
        const val MAX_COLUMNS = 10
        const val MIN_ROWS = 4
        const val MAX_ROWS = 10

        /** What a phone offers (a tablet, 600dp wide and up, gets the whole range). */
        const val PHONE_MAX_COLUMNS = 6
        const val PHONE_MIN_ROWS = 5
        const val PHONE_MAX_ROWS = 7
    }
}
