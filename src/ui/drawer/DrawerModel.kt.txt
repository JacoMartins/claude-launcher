package io.github.jacomartins.canyonui.ui.drawer

import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.compose.runtime.snapshotFlow
import io.github.jacomartins.canyonui.data.AppInfo
import io.github.jacomartins.canyonui.data.AppRepository
import io.github.jacomartins.canyonui.data.AppUsage
import io.github.jacomartins.canyonui.data.IconSize
import io.github.jacomartins.canyonui.data.SettingsStore
import io.github.jacomartins.canyonui.model.AppSearch
import io.github.jacomartins.canyonui.search.SearchModel
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.filterNotNull
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

/** Drawer state: search query, filtered apps, hidden apps and Canyon search's other results. */
class DrawerModel(
    context: android.content.Context,
    apps: AppRepository,
    private val settings: SettingsStore,
    private val scope: CoroutineScope,
) {
    /** Compose state so the text field updates synchronously while typing. */
    var query by mutableStateOf("")

    /** Bumped when a drag leaves the drawer, so open item menus close. */
    var dragEpoch by mutableIntStateOf(0)

    /** How far the Apps screen is slid down (px): its glass stays still against the wallpaper. */
    var offsetY by androidx.compose.runtime.mutableFloatStateOf(0f)

    /** An app is being moved (dragged out of the drawer): the letter dots hide meanwhile. */
    var moving by mutableStateOf(false)

    /**
     * The screen is Canyon search only (home-screen-only mode, where there is no app list): an
     * empty search shows suggestions and recent searches instead of every app.
     */
    var searchOnly by mutableStateOf(false)

    /** Bumped to put the cursor in the search field and show the keyboard. */
    var focusRequests by mutableIntStateOf(0)

    /** The most used apps, for an empty search. */
    var suggestions by mutableStateOf<List<AppInfo>>(emptyList())
        private set

    private val all = apps.apps.filterNotNull()
    private val hiddenKeys = settings.settings.map { it.hiddenApps }

    private val visible = combine(all, hiddenKeys) { list, hidden ->
        if (hidden.isEmpty()) list else list.filter { it.key.toString() !in hidden }
    }.stateIn(scope, SharingStarted.Eagerly, emptyList())

    // Worked out once per change of the app list, not on every key press: the accent-free
    // labels the filter compares, and the labels by package that search looks apps up in.
    private val normalized = visible.map { list -> list.map { it to AppSearch.normalize(it.label) } }
        .flowOn(Dispatchers.Default).stateIn(scope, SharingStarted.Eagerly, emptyList())
    private val labelsByPackage = visible.map { list -> list.associate { it.component.packageName to it.label } }
        .stateIn(scope, SharingStarted.Eagerly, emptyMap())

    /** Shortcuts, contacts, settings, calculator and hand-offs for the query. */
    val search = SearchModel(context, scope, snapshotFlow { query }) { labelsByPackage.value }

    val results: StateFlow<List<AppInfo>> = combine(normalized, snapshotFlow { query }) { list, q ->
        // "open camera" finds Camera.
        val appQ = io.github.jacomartins.canyonui.search.Commands.appQuery(q)
        if (appQ.isBlank()) list.map { it.first } else AppSearch.filterNormalized(list, appQ) { it.second }.map { it.first }
    }.flowOn(Dispatchers.Default).stateIn(scope, SharingStarted.Eagerly, emptyList())

    val hidden: StateFlow<List<AppInfo>> = combine(all, hiddenKeys) { list, hidden ->
        list.filter { it.key.toString() in hidden }
    }.stateIn(scope, SharingStarted.Eagerly, emptyList())

    /** The active layout's icon size; the Apps screen uses it too. */
    /** Bumped every minute when dynamic clock icons were redrawn. */
    val clockTicks: StateFlow<Int> = apps.clockTicks

    val iconSize: StateFlow<IconSize> = settings.settings.map { it.grid.iconSize }
        .stateIn(scope, SharingStarted.Eagerly, IconSize.MEDIUM)

    /** Before the screen opens: fresh suggestions, shortcuts and contacts permission. */
    fun prepare(usage: AppUsage) {
        suggestions = visible.value
            .map { it to usage.score(it.key.toString()) }
            .filter { it.second > 0 }
            .sortedByDescending { it.second }
            .take(SUGGESTIONS)
            .map { it.first }
        search.refresh()
    }

    /** Something was opened from the search: the query joins the recent searches. */
    fun rememberQuery() = search.remember(query)

    fun setHidden(app: AppInfo, hidden: Boolean) {
        scope.launch { settings.setHidden(app.key.toString(), hidden) }
    }

    private companion object {
        const val SUGGESTIONS = 8
    }
}
