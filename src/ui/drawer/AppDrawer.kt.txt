package io.github.jacomartins.canyonui.ui.drawer

import android.graphics.Rect
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.foundation.ExperimentalFoundationApi
import androidx.compose.foundation.Image
import androidx.compose.foundation.background
import androidx.compose.foundation.combinedClickable
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.interaction.collectIsFocusedAsState
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.asPaddingValues
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.ime
import androidx.compose.foundation.layout.navigationBars
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.statusBars
import androidx.compose.foundation.layout.union
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.layout.windowInsetsPadding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyGridScope
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.launch
import androidx.compose.foundation.lazy.grid.LazyGridState
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid
import androidx.compose.foundation.lazy.grid.items
import androidx.compose.foundation.lazy.grid.rememberLazyGridState
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.text.BasicTextField
import androidx.compose.foundation.text.KeyboardActions
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.runtime.snapshotFlow
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.draw.clip
import androidx.compose.ui.draw.drawWithContent
import androidx.compose.ui.focus.FocusRequester
import androidx.compose.ui.focus.focusRequester
import androidx.compose.ui.graphics.SolidColor
import androidx.compose.ui.hapticfeedback.HapticFeedbackType
import androidx.compose.ui.layout.LayoutCoordinates
import androidx.compose.ui.layout.boundsInWindow
import androidx.compose.ui.layout.onPlaced
import androidx.compose.ui.platform.LocalHapticFeedback
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.platform.LocalWindowInfo
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import io.github.jacomartins.canyonui.ui.home.AppIconView
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.ui.design.MenuEntry
import io.github.jacomartins.canyonui.ui.design.BackdropSource
import io.github.jacomartins.canyonui.ui.design.LocalBackdropHaze
import io.github.jacomartins.canyonui.ui.design.backdrop
import io.github.jacomartins.canyonui.ui.design.backdropColor
import dev.chrisbanes.haze.HazeState
import dev.chrisbanes.haze.hazeSource
import androidx.compose.runtime.CompositionLocalProvider
import io.github.jacomartins.canyonui.data.AppInfo
import androidx.compose.ui.draw.shadow
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.res.painterResource
import io.github.jacomartins.canyonui.ui.design.PrimaryButton
import io.github.jacomartins.canyonui.ui.design.SecondaryButton
import io.github.jacomartins.canyonui.ui.design.Spacing
import io.github.jacomartins.canyonui.ui.design.oneDepthBorder
import io.github.jacomartins.canyonui.ui.design.oneDepthTopBorder
import io.github.jacomartins.canyonui.ui.home.DragLayer
import kotlin.math.roundToInt

/** What the drawer asks the activity to do. */
interface DrawerActions {
    fun launch(app: AppInfo, bounds: Rect?)
    fun appInfo(app: AppInfo, bounds: Rect?)
    fun uninstall(app: AppInfo)
    fun editApp(app: AppInfo)

    /** Shows the shared long-press menu next to [anchor] (window coordinates). */
    fun showMenu(anchor: Rect, entries: List<MenuEntry>)
    fun addToHome(app: AppInfo)
    fun openSettings()

    /** Long-press in the drawer: moving the finger then drags the app onto the home screen. */
    fun armDrag(app: AppInfo, bounds: Rect?)

    /** Null when no speech recognizer is installed (the mic button is then hidden). */
    val voiceSearch: (() -> Unit)?

    // Canyon search's other results.
    fun openShortcut(shortcut: io.github.jacomartins.canyonui.search.ShortcutResult, bounds: Rect?)
    /** Opens another app's screen (settings page, contact, web search...); says so if none can. */
    fun openIntent(intent: android.content.Intent, bounds: Rect?)
    fun requestContacts()
    fun copyText(text: String)

    /** Calls [number] straight away once allowed (asks the first time; the dialer if refused). */
    fun placeCall(number: String)
}

private const val COLUMNS = 4

/**
 * The Apps screen's colour over what is behind it, by the Transparency setting: fairly solid
 * when translucent, a light tint over glass, and a medium one over the frosted blur.
 */
@Composable
private fun drawerTint(): Color {
    val blur by io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.active.collectAsState()
    val dark = io.github.jacomartins.canyonui.ui.design.isDarkSurface()
    val alpha = when (blur?.mode) {
        io.github.jacomartins.canyonui.data.RenderMode.GLASS -> if (dark) 0.32f else 0.2f
        io.github.jacomartins.canyonui.data.RenderMode.FROSTED -> if (dark) 0.62f else 0.5f
        else -> 0.9f
    }
    return MaterialTheme.colorScheme.surfaceContainerLow.copy(alpha = alpha)
}
private val SearchPillHeight = 56.dp

/** How long Go waits for the results of the text just typed. */
private const val GO_WAIT_MS = 1000L

/** The Apps screen's top corners while it slides (square when fully open). */
private val DRAWER_CORNER = 28.dp

/** The corner radius for the screen slid down by [offsetY]: full near the bottom, 0 at the top. */
private fun drawerCorner(offsetY: Float, height: Float, cornerPx: Float): Float =
    (offsetY / height.coerceAtLeast(1f) * 6f).coerceIn(0f, 1f) * cornerPx

/** The header row (options button) at the top of the Apps screen. */
private val TopBarHeight = 56.dp

@Composable
fun AppDrawer(model: DrawerModel, touch: DragLayer.DrawerTouchState, actions: DrawerActions) {
    val results by model.results.collectAsState()
    var showHidden by remember { mutableStateOf(false) }
    val gridState = rememberLazyGridState()
    val goScope = androidx.compose.runtime.rememberCoroutineScope()

    // The parent closes the drawer on a downward drag only when the grid is at its top.
    LaunchedEffect(Unit) {
        snapshotFlow { gridState.canScrollBackward }.collect { touch.contentAtTop = !it }
    }
    // A new query starts at the top. The list keeps its first visible row in place when rows
    // are inserted above it, so results that arrive a moment after the apps (actions, contacts,
    // shortcuts, the calculator, all placed above or among them) ended up above the screen. The
    // request to start at the top is now made after the new rows are in the list (this runs
    // once their composition is applied) and taken at the list's next layout.
    val extrasNow by model.search.extras.collectAsState()
    LaunchedEffect(model.query, results, extrasNow) {
        if (model.query.isNotBlank()) gridState.requestScrollToItem(0)
    }
    LaunchedEffect(model.query.isBlank()) { gridState.requestScrollToItem(0) }

    val bottomInsets = WindowInsets.navigationBars.union(WindowInsets.ime)

    val haze = remember { HazeState() }
    val cornerPx = with(LocalDensity.current) { DRAWER_CORNER.toPx() }
    CompositionLocalProvider(LocalBackdropHaze provides haze) {
    Box(
        Modifier
            .fillMaxSize()
            // Rounded top corners while the screen slides; square once it is all the way up.
            // Read in the layer only, so the slide redraws without recomposing.
            .graphicsLayer {
                val r = drawerCorner(model.offsetY, size.height, cornerPx)
                shape = androidx.compose.foundation.shape.RoundedCornerShape(topStart = r, topEnd = r)
                clip = r > 0.5f
            }
            // The interface border on the top edge and corners while they are rounded.
            .oneDepthTopBorder(DRAWER_CORNER) { h -> drawerCorner(model.offsetY, h, cornerPx) },
    ) {
        // Behind the apps, by the Transparency setting: a see-through colour, or the blurred
        // wallpaper (frosted) or glass over it, under a lighter tint.
        BackdropSource(haze) { model.offsetY }
        Box(
            Modifier
                .fillMaxSize()
                .backdrop(RoundedCornerShape(0.dp))
                .background(drawerTint()),
        )
        // The apps scroll under the header (it floats over them) instead of being cut off at it;
        // the letter dots stay between the header and the search field.
        val top = WindowInsets.statusBars.asPaddingValues().calculateTopPadding() + TopBarHeight
        // The apps are a Haze source too, so the glass search field bends the icons under it.
        Box(Modifier.fillMaxSize().hazeSource(haze, zIndex = 1f)) {
            val bottom = bottomInsets.asPaddingValues().calculateBottomPadding() + SearchPillHeight + 24.dp
            val searching = model.query.isNotBlank()
            val extras = extrasNow
            val askContacts by model.search.askContacts.collectAsState()
            val recents by model.search.recents.collectAsState()
            when {
                // Canyon search with nothing typed: suggested apps and recent searches.
                model.searchOnly && !searching -> AppGrid(
                    model.suggestions, gridState, top, bottom, actions, model,
                    header = { searchStart(model.suggestions.isNotEmpty()) },
                    footer = { searchRecents(recents, model) },
                )
                // Apps first, then every other kind of result (the calculator above them).
                searching -> AppGrid(
                    results, gridState, top, bottom, actions, model,
                    header = { searchTop(extras, model, actions, nothing = results.isEmpty() && extras.isEmpty) },
                    footer = { searchBottom(extras, askContacts, model, actions) },
                )
                else -> {
                    AppGrid(results, gridState, top, bottom, actions, model)
                    // The letter dots, for the full list.
                    if (!model.moving) {
                        AlphabetScrubber(results, gridState, touch, Modifier.align(Alignment.CenterEnd).padding(top = top, bottom = bottom))
                    }
                }
            }
        }
        Box(Modifier.fillMaxWidth().windowInsetsPadding(WindowInsets.statusBars)) {
            TopBar(onShowHidden = { showHidden = true }, onSettings = actions::openSettings, showMenu = actions::showMenu)
        }
        SearchPill(
            model = model,
            voiceSearch = actions.voiceSearch,
            onGo = { goScope.launch { openTop(model, actions) } },
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .windowInsetsPadding(bottomInsets)
                .padding(bottom = 12.dp),
        )
    }

    }

    if (showHidden) {
        HiddenAppsDialog(model, onDismiss = { showHidden = false })
    }
}

/** The keyboard's Go key: the first app, else what the query is (a number, a link), else the first other result, else the web. */
private suspend fun openTop(model: DrawerModel, actions: DrawerActions) {
    val q = model.query.trim()
    if (q.isEmpty()) return
    // The results for exactly this text (Go can come before they are worked out).
    val extras = kotlinx.coroutines.withTimeoutOrNull(GO_WAIT_MS) { model.search.extras.first { it.query == q } }
        ?: model.search.extras.value
    if (model.query.trim() != q) return
    model.rememberQuery()
    val app = model.results.value.firstOrNull()
    val current = extras.query == q
    when {
        current && extras.goFirst && extras.top.isNotEmpty() -> extras.top.first().let { model.search.record(it.key, it.person); runAction(it, null, actions) }
        app != null -> actions.launch(app, null)
        current && extras.top.isNotEmpty() -> extras.top.first().let { model.search.record(it.key, it.person); runAction(it, null, actions) }
        extras.direct.isNotEmpty() -> actions.openIntent(extras.direct.first().intent(), null)
        extras.shortcuts.isNotEmpty() -> actions.openShortcut(extras.shortcuts.first(), null)
        extras.contacts.isNotEmpty() -> actions.openIntent(android.content.Intent(android.content.Intent.ACTION_VIEW, extras.contacts.first().uri), null)
        else -> actions.openIntent(io.github.jacomartins.canyonui.search.HandOff(io.github.jacomartins.canyonui.search.HandOff.Kind.WEB, model.query.trim()).intent(), null)
    }
}

@Composable
private fun TopBar(onShowHidden: () -> Unit, onSettings: () -> Unit, showMenu: (Rect, List<MenuEntry>) -> Unit) {
    val coords = remember { Coords() }
    val hidden = stringResource(R.string.hidden_apps)
    val settings = stringResource(R.string.home_settings)
    Row(
        Modifier.fillMaxWidth().height(TopBarHeight).padding(horizontal = 8.dp, vertical = 4.dp),
        horizontalArrangement = Arrangement.End,
        verticalAlignment = Alignment.CenterVertically,
    ) {
        IconButton(
            onClick = {
                coords.bounds()?.let {
                    showMenu(it, listOf(MenuEntry(R.drawable.ph_eye, hidden, onShowHidden), MenuEntry(R.drawable.ph_settings, settings, onSettings)))
                }
            },
            modifier = Modifier.onPlaced { coords.value = it },
        ) {
            Icon(painterResource(R.drawable.ph_more), contentDescription = stringResource(R.string.drawer_options), tint = MaterialTheme.colorScheme.onSurface)
        }
    }
}

/**
 * Floating search field at the bottom. Compact and centred while browsing; widens to full width
 * when focused or holding a query. Sits above the keyboard.
 */
@Composable
private fun SearchPill(
    model: DrawerModel,
    voiceSearch: (() -> Unit)?,
    onGo: () -> Unit,
    modifier: Modifier = Modifier,
) {
    val colors = MaterialTheme.colorScheme
    val interaction = remember { MutableInteractionSource() }
    val focused by interaction.collectIsFocusedAsState()
    val expanded = focused || model.query.isNotEmpty()
    val widthFraction by animateFloatAsState(if (expanded) 1f else 0.66f, label = "searchWidth")
    val focus = remember { FocusRequester() }
    val searchLabel = stringResource(R.string.search)
    val placeholder = if (expanded) stringResource(R.string.search_everything) else searchLabel
    val keyboard = androidx.compose.ui.platform.LocalSoftwareKeyboardController.current
    LaunchedEffect(model.focusRequests) {
        if (model.focusRequests > 0) {
            focus.requestFocus()
            keyboard?.show()
        }
    }

    val blurOn = io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.active.collectAsState().value != null
    Surface(
        modifier = modifier
            .padding(horizontal = 16.dp)
            .fillMaxWidth(widthFraction)
            .height(SearchPillHeight)
            // No shadow under a see-through (blurred) field: it showed through and darkened it.
            .then(
                if (blurOn) Modifier
                else Modifier.shadow(Spacing.sm, CircleShape, ambientColor = Color.Black.copy(alpha = 0.25f), spotColor = Color.Black.copy(alpha = 0.375f)),
            )
            .oneDepthBorder()
            // Glass or frosted (by the Transparency setting) over the apps and the wallpaper.
            .backdrop(CircleShape, withContent = true),
        shape = CircleShape,
        color = backdropColor(colors.surfaceContainerHighest.copy(alpha = 0.92f)),
        onClick = { focus.requestFocus() },
    ) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(painterResource(R.drawable.ph_search), contentDescription = null, tint = colors.onSurfaceVariant, modifier = Modifier.padding(start = 20.dp, end = 12.dp).size(20.dp))
            BasicTextField(
                value = model.query,
                onValueChange = { model.query = it },
                singleLine = true,
                interactionSource = interaction,
                textStyle = MaterialTheme.typography.titleMedium.copy(color = colors.onSurface),
                cursorBrush = SolidColor(colors.primary),
                keyboardOptions = KeyboardOptions(imeAction = ImeAction.Go),
                keyboardActions = KeyboardActions(onGo = { onGo() }),
                modifier = Modifier
                    .weight(1f)
                    .focusRequester(focus)
                    .semantics { contentDescription = searchLabel },
                decorationBox = { inner ->
                    Box(contentAlignment = Alignment.CenterStart) {
                        if (model.query.isEmpty()) {
                            Text(placeholder, style = MaterialTheme.typography.titleMedium, color = colors.onSurfaceVariant, maxLines = 1, overflow = TextOverflow.Ellipsis)
                        }
                        inner()
                    }
                },
            )
            when {
                model.query.isNotEmpty() -> IconButton(onClick = { model.query = "" }) {
                    Icon(painterResource(R.drawable.ph_close), contentDescription = stringResource(R.string.clear_search), tint = colors.onSurfaceVariant, modifier = Modifier.size(20.dp))
                }
                voiceSearch != null -> IconButton(onClick = voiceSearch, modifier = Modifier.padding(end = 4.dp)) {
                    Icon(painterResource(R.drawable.ph_mic), contentDescription = stringResource(R.string.voice_search), tint = colors.onSurfaceVariant, modifier = Modifier.size(20.dp))
                }
                else -> Spacer(Modifier.width(20.dp))
            }
        }
    }
}

/** Holds an item's layout coordinates without causing recomposition when they change. */
private class Coords {
    var value: LayoutCoordinates? = null

    fun bounds(): Rect? {
        val c = value?.takeIf { it.isAttached } ?: return null
        val b = c.boundsInWindow()
        return Rect(b.left.roundToInt(), b.top.roundToInt(), b.right.roundToInt(), b.bottom.roundToInt())
    }
}

@OptIn(ExperimentalFoundationApi::class)
@Composable
private fun AppGrid(
    apps: List<AppInfo>,
    state: LazyGridState,
    top: androidx.compose.ui.unit.Dp,
    bottom: androidx.compose.ui.unit.Dp,
    actions: DrawerActions,
    model: DrawerModel,
    header: LazyGridScope.() -> Unit = {},
    footer: LazyGridScope.() -> Unit = {},
) {
    val optionsLabel = stringResource(R.string.action_options)
    val haptics = LocalHapticFeedback.current
    val addHome = stringResource(R.string.menu_add_to_home)
    val info = stringResource(R.string.menu_app_info)
    val editLabel = stringResource(R.string.menu_edit_app)
    val hide = stringResource(R.string.menu_hide)
    val uninstall = stringResource(R.string.menu_uninstall)
    fun appMenu(app: AppInfo, bounds: Rect) = buildList {
        add(MenuEntry(R.drawable.ph_plus, addHome) { actions.addToHome(app) })
        add(MenuEntry(R.drawable.ph_info, info) { actions.appInfo(app, bounds) })
        add(MenuEntry(R.drawable.ph_edit, editLabel) { actions.editApp(app) })
        add(MenuEntry(R.drawable.ph_hide, hide) { model.setHidden(app, true) })
        if (!app.isSystem) add(MenuEntry(R.drawable.ph_trash, uninstall) { actions.uninstall(app) })
    }
    // Same rule as the home screen: a share of the column width, capped.
    val size by model.iconSize.collectAsState()
    val windowWidthDp = with(LocalDensity.current) { LocalWindowInfo.current.containerSize.width.toDp().value }
    val iconDp = minOf((windowWidthDp - 16f) / COLUMNS * size.cellFraction, size.maxDp.toFloat()).dp
    val labelStyle = MaterialTheme.typography.bodySmall.copy(fontSize = AppIconView.LABEL_SP.sp, lineHeight = (AppIconView.LABEL_SP + 2).sp)
    // Read only inside the draw phase of clock icons, so a tick repaints them without recomposing.
    val clockTick = model.clockTicks.collectAsState()
    LazyVerticalGrid(
        columns = GridCells.Fixed(COLUMNS),
        state = state,
        // Room at the end for the letter dots.
        contentPadding = PaddingValues(start = 8.dp, end = 24.dp, top = top + 8.dp, bottom = bottom),
        modifier = Modifier.fillMaxSize(),
    ) {
        header()
        items(apps, key = { it.key.toString() }, contentType = { 0 }) { app ->
            val coords = remember { Coords() }
            Box {
                Column(
                    Modifier
                        .fillMaxWidth()
                        .onPlaced { coords.value = it }
                        .clip(RoundedCornerShape(20.dp))
                        .combinedClickable(
                            role = Role.Button,
                            onLongClickLabel = optionsLabel,
                            onLongClick = {
                                haptics.performHapticFeedback(HapticFeedbackType.LongPress)
                                coords.bounds()?.let { b -> actions.showMenu(b, appMenu(app, b)) }
                                actions.armDrag(app, coords.bounds())
                            },
                            onClick = {
                                model.rememberQuery()
                                actions.launch(app, coords.bounds())
                            },
                        )
                        .padding(vertical = 12.dp),
                    horizontalAlignment = Alignment.CenterHorizontally,
                ) {
                    Image(
                        app.image,
                        contentDescription = null,
                        modifier = Modifier
                            .size(iconDp)
                            // A clock's bitmap is redrawn in place each minute; reading the tick
                            // here repaints just this icon.
                            .then(if (app.isClock) Modifier.drawWithContent { clockTick.value; drawContent() } else Modifier),
                    )
                    Spacer(Modifier.height(AppIconView.LABEL_GAP_DP.dp))
                    Text(
                        app.label,
                        style = labelStyle,
                        color = MaterialTheme.colorScheme.onSurface,
                        maxLines = 1,
                        overflow = TextOverflow.Ellipsis,
                        textAlign = TextAlign.Center,
                        modifier = Modifier.padding(horizontal = 4.dp),
                    )
                }
            }
        }
        footer()
    }
}

@Composable
fun HiddenAppsDialog(model: DrawerModel, onDismiss: () -> Unit) {
    val hidden by model.hidden.collectAsState()
    io.github.jacomartins.canyonui.ui.design.SheetDialog(
        onDismissRequest = onDismiss,
        confirmButton = { PrimaryButton(stringResource(R.string.done), onDismiss) },
        title = { Text(stringResource(R.string.hidden_apps)) },
        text = {
            if (hidden.isEmpty()) {
                Text(stringResource(R.string.hidden_apps_empty))
            } else {
                LazyColumn {
                    items(hidden, key = { it.key.toString() }) { app ->
                        Row(Modifier.fillMaxWidth().padding(vertical = 4.dp), verticalAlignment = Alignment.CenterVertically) {
                            Image(app.image, contentDescription = null, modifier = Modifier.size(36.dp))
                            Spacer(Modifier.width(12.dp))
                            Text(app.label, Modifier.weight(1f), maxLines = 1, overflow = TextOverflow.Ellipsis)
                            SecondaryButton(stringResource(R.string.show), onClick = { model.setHidden(app, false) }, icon = R.drawable.ph_eye)
                        }
                    }
                }
            }
        },
    )
}
