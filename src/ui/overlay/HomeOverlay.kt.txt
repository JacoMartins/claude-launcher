package io.github.jacomartins.canyonui.ui.overlay

import android.graphics.Rect
import androidx.annotation.DrawableRes
import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.animation.scaleIn
import androidx.compose.animation.scaleOut
import androidx.compose.animation.slideInVertically
import androidx.compose.animation.slideOutVertically
import androidx.compose.foundation.ExperimentalFoundationApi
import androidx.compose.foundation.Image
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.combinedClickable
import androidx.compose.foundation.gestures.detectHorizontalDragGestures
import androidx.compose.foundation.gestures.detectDragGesturesAfterLongPress
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.IntrinsicSize
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.BoxWithConstraints
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.aspectRatio
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.ime
import androidx.compose.foundation.layout.navigationBars
import androidx.compose.foundation.layout.offset
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.safeDrawing
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.statusBars
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.layout.widthIn
import androidx.compose.foundation.layout.windowInsetsPadding
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid
import androidx.compose.foundation.lazy.grid.itemsIndexed
import androidx.compose.foundation.pager.HorizontalPager
import androidx.compose.foundation.pager.rememberPagerState
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.text.BasicTextField
import androidx.compose.foundation.text.KeyboardActions
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableFloatStateOf
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.animation.core.FastOutSlowInEasing
import androidx.compose.animation.core.animateFloat
import androidx.compose.animation.EnterExitState
import androidx.compose.ui.graphics.TransformOrigin
import androidx.compose.animation.core.tween
import androidx.compose.animation.togetherWith
import androidx.compose.animation.AnimatedContent
import androidx.compose.animation.EnterTransition
import androidx.compose.animation.ExitTransition
import androidx.compose.ui.util.lerp
import androidx.compose.ui.draw.drawBehind
import androidx.compose.ui.draw.clip
import androidx.compose.ui.draw.shadow
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.Color
import io.github.jacomartins.canyonui.ui.design.ColorPicker
import androidx.compose.ui.graphics.Brush
import androidx.compose.ui.graphics.Shadow
import androidx.compose.ui.graphics.SolidColor
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.hapticfeedback.HapticFeedbackType
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.layout.LayoutCoordinates
import androidx.compose.ui.layout.boundsInWindow
import androidx.compose.ui.layout.layout
import androidx.compose.ui.layout.onGloballyPositioned
import androidx.compose.ui.layout.onPlaced
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.platform.LocalHapticFeedback
import androidx.compose.ui.res.painterResource
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.IntOffset
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.Dp
import androidx.compose.material3.LocalMinimumInteractiveComponentSize
import androidx.compose.runtime.CompositionLocalProvider
import androidx.compose.ui.unit.sp
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.data.AppEntry
import io.github.jacomartins.canyonui.data.FolderEntry
import io.github.jacomartins.canyonui.ui.design.IconPillButton
import io.github.jacomartins.canyonui.ui.design.backdrop
import io.github.jacomartins.canyonui.ui.design.BackdropSource
import io.github.jacomartins.canyonui.ui.design.LocalBackdropHaze
import dev.chrisbanes.haze.HazeState
import io.github.jacomartins.canyonui.ui.design.backdropColor
import io.github.jacomartins.canyonui.ui.design.PrimaryButton
import io.github.jacomartins.canyonui.ui.design.SecondaryButton
import io.github.jacomartins.canyonui.ui.design.MenuLayer
import io.github.jacomartins.canyonui.ui.design.Spacing
import io.github.jacomartins.canyonui.ui.design.oneDepthBorder
import io.github.jacomartins.canyonui.ui.home.AppIconView
import kotlinx.coroutines.delay
import kotlin.math.roundToInt

/** Folder colours offered in the folder popup (0 = theme default). */
val FolderColors = listOf(0, 0xA65B8DEF.toInt(), 0xA63FB27F.toInt(), 0xA6F2B63D.toInt(), 0xA6F0805A.toInt(), 0xA6E86A8F.toInt(), 0xA69B7BE8.toInt())

private val PanelShape = RoundedCornerShape(32.dp)

@Composable
fun HomeOverlay(state: OverlayState, actions: OverlayActions) {
    val haze = remember { HazeState() }
    androidx.compose.runtime.CompositionLocalProvider(LocalBackdropHaze provides haze) {
    Box(Modifier.fillMaxSize()) {
        // Frosted and glass: the wallpaper copy the panels' Haze effects sample, or, behind an
        // open folder, a copy of the home screen (wallpaper, icons and widgets) so they show
        // through the panel. It is dropped once the folder has faded out.
        BackdropSource(haze, content = state.folderBackdrop)
        androidx.compose.runtime.LaunchedEffect(state.folder == null) {
            if (state.folder == null) {
                kotlinx.coroutines.delay(CLOSE_MS.toLong())
                state.folderBackdrop = null
            }
        }
        val chrome = state.editMode && state.folder == null && state.picker == null && !state.dragging
        AnimatedVisibility(
            visible = chrome,
            enter = fadeIn() + slideInVertically { -it / 2 },
            exit = fadeOut() + slideOutVertically { -it / 2 },
            modifier = Modifier.align(Alignment.TopCenter),
        ) { EditTopBar(state, actions) }
        AnimatedVisibility(
            visible = chrome,
            enter = fadeIn() + slideInVertically { it / 2 },
            exit = fadeOut() + slideOutVertically { it / 2 },
            modifier = Modifier.align(Alignment.BottomCenter),
        ) {
            EditBottomBar(actions)
        }

        state.picker?.let { WidgetPicker(it, actions) }
        // Folder: the dim fades in and out, and the panel grows out of the folder's tile on the
        // home screen (and shrinks back into it). No size animation of the container: that
        // revealed the dim from a corner.
        AnimatedContent(
            targetState = state.folder,
            contentKey = { it?.id },
            transitionSpec = { EnterTransition.None togetherWith ExitTransition.None using null },
            label = "folder",
        ) { f ->
            if (f != null) {
                val open by transition.animateFloat(
                    transitionSpec = { tween(if (targetState == EnterExitState.Visible) OPEN_MS else CLOSE_MS, easing = FastOutSlowInEasing) },
                    label = "folderOpen",
                ) { if (it == EnterExitState.Visible) 1f else 0f }
                FolderPopup(f, state, actions, open = { open })
            }
        }
        // Above an open folder: the app being edited may be inside it.
        state.editApp?.let { EditAppSheet(it, actions) }
        AnimatedContent(
            targetState = state.menu,
            transitionSpec = { fadeIn(tween(MENU_MS)) togetherWith fadeOut(tween(MENU_MS)) using null },
            label = "menu",
        ) { m ->
            if (m != null) {
                // Zooms out of the pressed item, and back into it.
                val open by transition.animateFloat(
                    transitionSpec = { tween(MENU_MS, easing = FastOutSlowInEasing) },
                    label = "menuOpen",
                ) { if (it == EnterExitState.Visible) 1f else 0f }
                MenuLayer(m.anchor, m.entries, { state.menu = null }, progress = { open })
            }
        }
        state.confirmDeletePage?.let { DeletePageDialog(it, state, actions) }
        if (state.confirmCategorize) CategorizeDialog(state, actions)
        SnackHost(state, Modifier.align(Alignment.BottomCenter))
    }
    }
}

// ------------------------------------------------------------------------------- edit mode

/**
 * Just above the focused page in edit mode: make it the home page (left; a filled house when it
 * already is) and delete it (right). Only the two buttons are drawn and take touches; the row
 * between them is empty space. Hidden on the plus page at the end.
 */
@Composable
private fun EditTopBar(state: OverlayState, actions: OverlayActions) {
    val page = state.currentPage
    if (page >= state.pageCount) return
    val isHome = page == HOME_PAGE
    val density = LocalDensity.current
    val minTop = WindowInsets.statusBars.getTop(density)
    val gap = with(density) { Spacing.sm.roundToPx() }
    Row(
        Modifier
            .fillMaxWidth(state.editScale)
            .layout { measurable, constraints ->
                val p = measurable.measure(constraints)
                // Bottom edge a little above the page; never under the status bar.
                val y = (state.editPageTop - p.height - gap).coerceAtLeast(minTop + gap)
                layout(p.width, y + p.height) { p.place(0, y) }
            },
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically,
    ) {
        PageIconButton(
            icon = if (isHome) R.drawable.ph_house_fill else R.drawable.ph_house,
            label = stringResource(if (isHome) R.string.home_page else R.string.make_home_page),
            enabled = !isHome,
        ) { actions.makeHomePage(page) }
        // Category mode: file the loose apps on the pages after the home page. Styled like
        // the two page buttons beside it.
        if (state.categoryMode) {
            SecondaryButton(stringResource(R.string.categorize_apps), { state.confirmCategorize = true }, icon = R.drawable.ph_stack)
        }
        PageIconButton(R.drawable.ph_trash, stringResource(R.string.delete_page), enabled = !isHome && state.pageCount > 1) {
            state.confirmDeletePage = page
        }
    }
}

/** A round secondary button, icon only. */
@Composable
private fun PageIconButton(@DrawableRes icon: Int, label: String, enabled: Boolean, onClick: () -> Unit) =
    IconPillButton(icon, label, onClick, enabled = enabled)

@Composable
private fun EditBottomBar(actions: OverlayActions) {
    Surface(
        modifier = Modifier
            .windowInsetsPadding(WindowInsets.navigationBars)
            .padding(bottom = Spacing.lg)
            .shadow(Spacing.sm, PanelShape, ambientColor = Color.Black.copy(alpha = 0.25f), spotColor = Color.Black.copy(alpha = 0.375f))
            .oneDepthBorder(cornerRadius = 32.dp)
            .backdrop(PanelShape),
        shape = PanelShape,
        color = backdropColor(MaterialTheme.colorScheme.surfaceContainer.copy(alpha = 0.94f)),
    ) {
        Row(Modifier.padding(Spacing.sm), horizontalArrangement = Arrangement.spacedBy(Spacing.xs)) {
            ToolButton(R.drawable.ph_wallpaper, stringResource(R.string.wallpaper), actions::chooseWallpaper)
            ToolButton(R.drawable.ph_widgets, stringResource(R.string.widgets), actions::openWidgets)
            ToolButton(R.drawable.ph_settings, stringResource(R.string.settings), actions::openSettings)
        }
    }
}

@Composable
private fun ToolButton(@DrawableRes icon: Int, label: String, onClick: () -> Unit) {
    Column(
        Modifier
            .widthIn(min = 72.dp)
            .clip(RoundedCornerShape(24.dp))
            .clickable(role = Role.Button, onClick = onClick)
            .padding(horizontal = Spacing.sm, vertical = Spacing.sm),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(Spacing.xsSm),
    ) {
        Surface(shape = CircleShape, color = MaterialTheme.colorScheme.secondaryContainer, modifier = Modifier.size(48.dp)) {
            Box(contentAlignment = Alignment.Center) {
                Icon(painterResource(icon), contentDescription = null, tint = MaterialTheme.colorScheme.onSecondaryContainer, modifier = Modifier.size(22.dp))
            }
        }
        Text(label, style = MaterialTheme.typography.labelLarge, color = MaterialTheme.colorScheme.onSurface)
    }
}

// ------------------------------------------------------------------------------- long-press menu



// ------------------------------------------------------------------------------- folder popup

private class Coords {
    var value: LayoutCoordinates? = null

    fun bounds(): Rect? {
        val c = value?.takeIf { it.isAttached } ?: return null
        val b = c.boundsInWindow()
        return Rect(b.left.roundToInt(), b.top.roundToInt(), b.right.roundToInt(), b.bottom.roundToInt())
    }
}

/** An open folder is always a 3x3 square of apps, page after page. */
private const val FOLDER_COLUMNS = 3
private const val FOLDER_PER_PAGE = 9

@OptIn(ExperimentalFoundationApi::class)
@Composable
private fun FolderPopup(folder: FolderEntry, state: OverlayState, actions: OverlayActions, open: () -> Float) {
    var title by remember(folder.id) { mutableStateOf(folder.title) }
    var showColors by remember(folder.id) { mutableStateOf(false) }
    var showPicker by remember(folder.id) { mutableStateOf(false) }
    val pickerLabel = stringResource(R.string.folder_custom_color)
    fun close() {
        if (title.trim() != folder.title) actions.renameFolder(folder, title)
        actions.closeFolder()
    }
    Box(
        Modifier
            .fillMaxSize()
            .drawBehind { drawRect(Color.Black.copy(alpha = FOLDER_DIM * open())) }
            .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null) { close() }
            .windowInsetsPadding(WindowInsets.ime),
        contentAlignment = Alignment.Center,
    ) {
        // The name and colour choice float above the panel, over the dimmed home screen; the
        // panel holds only the apps.
        Column(Modifier.padding(horizontal = Spacing.md).fillMaxWidth(), horizontalAlignment = Alignment.CenterHorizontally) {
            Row(Modifier.padding(horizontal = Spacing.lg).graphicsLayer { alpha = open() }, verticalAlignment = Alignment.CenterVertically) {
                val nameLabel = stringResource(R.string.folder_name)
                BasicTextField(
                    value = title,
                    onValueChange = { title = it.take(40) },
                    singleLine = true,
                    textStyle = MaterialTheme.typography.headlineMedium.copy(color = Color.White, textAlign = TextAlign.Center, shadow = TitleShadow),
                    cursorBrush = SolidColor(MaterialTheme.colorScheme.primary),
                    keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
                    keyboardActions = KeyboardActions(onDone = { if (title.trim() != folder.title) actions.renameFolder(folder, title) }),
                    modifier = Modifier.weight(1f).semantics { contentDescription = nameLabel },
                    decorationBox = { inner ->
                        Box(contentAlignment = Alignment.Center) {
                            if (title.isEmpty()) Text(stringResource(R.string.folder_default_title), style = MaterialTheme.typography.headlineMedium.copy(shadow = TitleShadow), color = Color.White.copy(alpha = 0.7f))
                            inner()
                        }
                    },
                )
                Box(
                    Modifier
                        .size(40.dp)
                        .clip(CircleShape)
                        .clickable(role = Role.Button) { showColors = !showColors },
                    contentAlignment = Alignment.Center,
                ) {
                    Icon(painterResource(R.drawable.ph_palette), contentDescription = stringResource(R.string.folder_color), tint = Color.White)
                }
            }
            AnimatedVisibility(visible = showColors) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Row(Modifier.padding(top = Spacing.smMd), horizontalArrangement = Arrangement.spacedBy(Spacing.sm)) {
                        for (c in FolderColors) {
                            val selected = folder.color == c
                            Box(
                                Modifier
                                    .size(32.dp)
                                    .clip(CircleShape)
                                    .background(if (c == 0) MaterialTheme.colorScheme.surfaceVariant else Color(c))
                                    .border(if (selected) 2.dp else 0.dp, MaterialTheme.colorScheme.primary, CircleShape)
                                    .clickable(role = Role.RadioButton) { actions.setFolderColor(folder, c) },
                                contentAlignment = Alignment.Center,
                            ) {
                                if (selected) Icon(painterResource(R.drawable.ph_check), contentDescription = null, modifier = Modifier.size(16.dp), tint = if (c == 0) MaterialTheme.colorScheme.onSurfaceVariant else Color.White)
                            }
                        }
                        // Any colour, with transparency.
                        val custom = folder.color != 0 && folder.color !in FolderColors
                        Box(
                            Modifier
                                .size(32.dp)
                                .clip(CircleShape)
                                .background(Brush.sweepGradient(RainbowColors))
                                .border(if (custom || showPicker) 2.dp else 0.dp, MaterialTheme.colorScheme.primary, CircleShape)
                                .clickable(role = Role.Button, onClickLabel = stringResource(R.string.folder_custom_color)) { showPicker = !showPicker }
                                .semantics { contentDescription = pickerLabel },
                        )
                    }
                    AnimatedVisibility(visible = showPicker) {
                        Surface(
                            shape = RoundedCornerShape(24.dp),
                            color = backdropColor(MaterialTheme.colorScheme.surfaceContainer.copy(alpha = 0.97f)),
                            modifier = Modifier
                                .padding(top = Spacing.smMd)
                                .fillMaxWidth()
                                .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null) { }
                                .oneDepthBorder(cornerRadius = 24.dp)
                                .backdrop(RoundedCornerShape(24.dp)),
                        ) {
                            ColorPicker(
                                initial = folder.color,
                                labels = listOf(
                                    stringResource(R.string.color_hue), stringResource(R.string.color_saturation),
                                    stringResource(R.string.color_brightness), stringResource(R.string.color_opacity),
                                ),
                                onCommit = { actions.setFolderColor(folder, it) },
                                modifier = Modifier.padding(Spacing.md),
                            )
                        }
                    }
                }
            }
            Spacer(Modifier.height(Spacing.md))

            val iconSize = state.homeIconDp.dp
            val rowHeight = Spacing.smMd * 2 + iconSize + AppIconView.LABEL_GAP_DP.dp +
                with(LocalDensity.current) { (AppIconView.LABEL_SP + 3).sp.toDp() }
            // Where the panel ends up (window px), measured outside its own transform.
            var bounds by remember { mutableStateOf(androidx.compose.ui.geometry.Rect.Zero) }
            Surface(
                modifier = Modifier
                    .onGloballyPositioned { c -> bounds = c.boundsInWindow() }
                    // Grows out of the folder's tile: from its place and size to the panel's.
                    .graphicsLayer {
                        val t = open()
                        val from = state.folderFrom
                        if (from == null || bounds.isEmpty) {
                            alpha = t
                            return@graphicsLayer
                        }
                        scaleX = lerp(from.width / bounds.width, 1f, t)
                        scaleY = lerp(from.height / bounds.height, 1f, t)
                        translationX = (from.center.x - bounds.center.x) * (1f - t)
                        translationY = (from.center.y - bounds.center.y) * (1f - t)
                        alpha = (t * 4f).coerceIn(0f, 1f)
                    }
                    // Square cells: each column is as wide as a row is tall.
                    .width(rowHeight * FOLDER_COLUMNS + Spacing.sm * 2)
                    .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null) { } // keep taps inside
                    .oneDepthBorder(cornerRadius = 32.dp)
                    .backdrop(PanelShape),
                shape = PanelShape,
                // More see-through than the other panels: the apps float over the glass or blur.
                color = backdropColor(MaterialTheme.colorScheme.surfaceContainer.copy(alpha = FOLDER_PANEL_ALPHA)),
            ) {
                // The apps appear once the panel has grown most of the way.
                Column(
                    Modifier
                        .graphicsLayer { alpha = ((open() - 0.35f) / 0.65f).coerceIn(0f, 1f) }
                        .padding(vertical = Spacing.md),
                    horizontalAlignment = Alignment.CenterHorizontally,
                ) {
                    val pages = (folder.apps.size + FOLDER_PER_PAGE - 1) / FOLDER_PER_PAGE
                    val pager = rememberPagerState { pages.coerceAtLeast(1) }
                    // Always three rows of three square cells (even with fewer apps), with icons at
                    // the home screen's size, so every folder opens as the same square.
                    val rowsPerPage = FOLDER_PER_PAGE / FOLDER_COLUMNS
                    HorizontalPager(state = pager, modifier = Modifier.fillMaxWidth()) { page ->
                        val apps = folder.apps.drop(page * FOLDER_PER_PAGE).take(FOLDER_PER_PAGE)
                        Column(
                            Modifier
                                .padding(horizontal = Spacing.sm)
                                .height(rowHeight * rowsPerPage)
                                // Where a dragged app lands when it is dropped back in the folder.
                                .onGloballyPositioned { c ->
                                    if (page == pager.currentPage) {
                                        val b = c.boundsInWindow()
                                        state.folderGrid.set(b.left, b.top, b.right, b.bottom)
                                        state.folderPage = page
                                    }
                                },
                            verticalArrangement = Arrangement.Top,
                        ) {
                            for (row in apps.chunked(FOLDER_COLUMNS)) {
                                Row(Modifier.fillMaxWidth().height(rowHeight)) {
                                    for (app in row) {
                                        // The app being dragged leaves a gap where it was.
                                        val m = Modifier.weight(1f).graphicsLayer { alpha = if (state.folderDragId == app.id) 0f else 1f }
                                        FolderApp(app, folder, actions, iconSize, m)
                                    }
                                    repeat(FOLDER_COLUMNS - row.size) { Spacer(Modifier.weight(1f)) }
                                }
                            }
                        }
                    }
                    if (pages > 1) {
                        // Like the home screen's: filled circles, the page in view opaque, the
                        // others faint, following the swipe.
                        Row(Modifier.padding(top = Spacing.sm), horizontalArrangement = Arrangement.spacedBy(7.dp)) {
                            repeat(pages) { i ->
                                Icon(
                                    painterResource(R.drawable.ph_circle_fill),
                                    contentDescription = null,
                                    tint = MaterialTheme.colorScheme.onSurface,
                                    modifier = Modifier
                                        .size(9.dp)
                                        .graphicsLayer {
                                            val pos = pager.currentPage + pager.currentPageOffsetFraction
                                            val t = (1f - kotlin.math.abs(pos - i)).coerceIn(0f, 1f)
                                            alpha = 0.35f + 0.65f * t
                                        },
                                )
                            }
                        }
                    }
                }
            }
        }
    }
}

private val RainbowColors = listOf(
    Color(0xFFFF0000), Color(0xFFFFFF00), Color(0xFF00FF00), Color(0xFF00FFFF),
    Color(0xFF0000FF), Color(0xFFFF00FF), Color(0xFFFF0000),
)

/** Keeps white text readable over a light wallpaper behind the dimmed backdrop. */
private val TitleShadow = Shadow(Color.Black.copy(alpha = 0.45f), Offset(0f, 2f), 8f)

@OptIn(ExperimentalFoundationApi::class)
@Composable
private fun FolderApp(app: AppEntry, folder: FolderEntry, actions: OverlayActions, iconSize: androidx.compose.ui.unit.Dp, modifier: Modifier) {
    val coords = remember { Coords() }
    val haptics = LocalHapticFeedback.current
    Column(
        modifier
            .onPlaced { coords.value = it }
            .clip(RoundedCornerShape(20.dp))
            .combinedClickable(
                role = Role.Button,
                onClick = { actions.launchFromFolder(app, coords.bounds()) },
                onLongClick = {
                    haptics.performHapticFeedback(HapticFeedbackType.LongPress)
                    coords.bounds()?.let { actions.armDragFromFolder(app, folder, it) }
                },
            )
            .padding(vertical = Spacing.smMd),
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        Image(app.app.image, contentDescription = null, modifier = Modifier.size(iconSize))
        Spacer(Modifier.height(AppIconView.LABEL_GAP_DP.dp))
        Text(
            app.app.label,
            style = MaterialTheme.typography.bodySmall.copy(fontSize = AppIconView.LABEL_SP.sp, lineHeight = (AppIconView.LABEL_SP + 3).sp),
            color = MaterialTheme.colorScheme.onSurface,
            maxLines = 1,
            overflow = TextOverflow.Ellipsis,
            textAlign = TextAlign.Center,
            modifier = Modifier.padding(horizontal = Spacing.xs),
        )
    }
}

// ------------------------------------------------------------------------------- snackbar

@Composable
private fun SnackHost(state: OverlayState, modifier: Modifier) {
    val snack = state.snack
    LaunchedEffect(snack?.id) {
        if (snack != null) {
            delay(SNACK_MS)
            if (state.snack === snack) state.snack = null
        }
    }
    AnimatedVisibility(
        visible = snack != null,
        enter = fadeIn() + scaleIn(initialScale = 0.92f),
        exit = fadeOut() + scaleOut(targetScale = 0.92f),
        modifier = modifier,
    ) {
        val s = snack ?: return@AnimatedVisibility
        // Swiped sideways far enough, the message goes away early.
        var dx by remember(s.id) { mutableFloatStateOf(0f) }
        val dismissPx = with(LocalDensity.current) { 96.dp.toPx() }
        Surface(
            modifier = Modifier
                .graphicsLayer {
                    translationX = dx
                    alpha = (1f - kotlin.math.abs(dx) / (dismissPx * 1.5f)).coerceIn(0f, 1f)
                }
                .pointerInput(s.id) {
                    detectHorizontalDragGestures(
                        onDragEnd = { if (kotlin.math.abs(dx) > dismissPx) state.snack = null else dx = 0f },
                        onDragCancel = { dx = 0f },
                    ) { change, amount -> change.consume(); dx += amount }
                }
                .windowInsetsPadding(WindowInsets.navigationBars)
                .padding(bottom = 128.dp, start = Spacing.md, end = Spacing.md)
                .shadow(Spacing.sm, PanelShape, ambientColor = Color.Black.copy(alpha = 0.25f), spotColor = Color.Black.copy(alpha = 0.375f))
                .oneDepthBorder(cornerRadius = 32.dp)
                .backdrop(PanelShape),
            shape = PanelShape,
            // The mode's own surface (not the inverted one), like every other panel.
            color = backdropColor(MaterialTheme.colorScheme.surfaceContainerHigh.copy(alpha = 0.97f)),
        ) {
            Row(Modifier.padding(start = Spacing.mdLg, end = Spacing.xs, top = Spacing.xs, bottom = Spacing.xs), verticalAlignment = Alignment.CenterVertically) {
                Text(s.message, style = MaterialTheme.typography.bodyMedium, color = MaterialTheme.colorScheme.onSurface, modifier = Modifier.padding(end = Spacing.md))
                val undo = s.onUndo
                if (undo != null) PrimaryButton(stringResource(R.string.undo), { state.snack = null; undo() })
                IconPillButton(R.drawable.ph_close, stringResource(R.string.dismiss), { state.snack = null }, Modifier.padding(start = Spacing.xs), size = 40.dp)
            }
        }
    }
}

/** Asks before the loose apps are filed into category folders. */
@Composable
private fun CategorizeDialog(state: OverlayState, actions: OverlayActions) {
    androidx.compose.material3.AlertDialog(
        onDismissRequest = { state.confirmCategorize = false },
        title = { Text(stringResource(R.string.categorize_title)) },
        text = { Text(stringResource(R.string.categorize_message)) },
        confirmButton = {
            PrimaryButton(stringResource(R.string.categorize_apps), {
                state.confirmCategorize = false
                actions.categorizeApps()
            })
        },
        dismissButton = { SecondaryButton(stringResource(R.string.cancel), { state.confirmCategorize = false }) },
        shape = RoundedCornerShape(28.dp),
    )
}

/** Asks before a page is deleted (with everything on it). */
@Composable
private fun DeletePageDialog(page: Int, state: OverlayState, actions: OverlayActions) {
    androidx.compose.material3.AlertDialog(
        onDismissRequest = { state.confirmDeletePage = null },
        title = { Text(stringResource(R.string.delete_page_title)) },
        text = { Text(stringResource(R.string.delete_page_message)) },
        confirmButton = {
            PrimaryButton(stringResource(R.string.delete), {
                state.confirmDeletePage = null
                actions.deletePage(page)
            })
        },
        dismissButton = { SecondaryButton(stringResource(R.string.cancel), { state.confirmDeletePage = null }) },
        shape = RoundedCornerShape(28.dp),
    )
}

private const val SNACK_MS = 5000L

/** The open folder's panel colour (before the Transparency effect thins it further). */
private const val FOLDER_PANEL_ALPHA = 0.72f

/** How dark the home screen gets behind an open folder. */
private const val FOLDER_DIM = 0.35f
private const val OPEN_MS = 320
private const val CLOSE_MS = 260
private const val MENU_MS = 220

/** The first page is the home page (the house in the page indicator); it cannot be deleted. */
private const val HOME_PAGE = 0
