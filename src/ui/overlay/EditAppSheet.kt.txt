package io.github.jacomartins.canyonui.ui.overlay

import androidx.compose.foundation.Image
import androidx.compose.foundation.background
import androidx.compose.foundation.clickable
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.aspectRatio
import androidx.compose.foundation.layout.fillMaxHeight
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.ui.draw.rotate
import androidx.compose.foundation.layout.heightIn
import androidx.compose.foundation.layout.imePadding
import androidx.compose.foundation.layout.navigationBars
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.windowInsetsPadding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid
import androidx.compose.foundation.lazy.grid.items
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.text.BasicTextField
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.produceState
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import io.github.jacomartins.canyonui.ui.design.SectionHeader
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.SolidColor
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.platform.LocalConfiguration
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.res.painterResource
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.dp
import androidx.core.graphics.drawable.toBitmap
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.data.AppInfo
import io.github.jacomartins.canyonui.data.IconFiles
import io.github.jacomartins.canyonui.data.IconOverride
import io.github.jacomartins.canyonui.data.IconPack
import io.github.jacomartins.canyonui.data.IconPackInfo
import io.github.jacomartins.canyonui.data.titleRes
import io.github.jacomartins.canyonui.model.AppCategories
import io.github.jacomartins.canyonui.ui.design.PrimaryButton
import io.github.jacomartins.canyonui.ui.design.SecondaryButton
import io.github.jacomartins.canyonui.ui.design.Spacing
import io.github.jacomartins.canyonui.ui.design.oneDepthBorder
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.io.File

/**
 * Editing one app's label and icon. [draft] is what will be saved; the activity updates it when
 * an image comes back from the photo picker.
 */
/** [showCategory]: "Organize apps by category" is on, so the app's category can be chosen. */
class EditAppState(val app: AppInfo, val original: IconOverride?, val packs: List<IconPackInfo>, val showCategory: Boolean = false) {
    var draft by mutableStateOf(original ?: IconOverride())
}

private val SheetShape = RoundedCornerShape(topStart = 32.dp, topEnd = 32.dp)
private const val PREVIEW_PX = 192

@Composable
fun EditAppSheet(state: EditAppState, actions: OverlayActions) {
    val colors = MaterialTheme.colorScheme
    var label by remember(state) { mutableStateOf(state.original?.label ?: "") }
    // Null: the main page; else the pack whose icons are being browsed.
    var browsing by remember(state) { mutableStateOf<IconPackInfo?>(null) }
    var choosingPack by remember(state) { mutableStateOf(false) }
    fun close() = actions.closeEditApp()

    Box(
        Modifier
            .fillMaxSize()
            .background(colors.scrim.copy(alpha = 0.4f))
            .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null, onClick = ::close),
    ) {
        Surface(
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .fillMaxWidth()
                .then(if (browsing != null) Modifier.fillMaxHeight(0.9f) else Modifier)
                .oneDepthBorder(cornerRadius = 32.dp)
                .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null) {},
            shape = SheetShape,
            color = colors.surfaceContainer,
        ) {
            Column(Modifier.windowInsetsPadding(WindowInsets.navigationBars).imePadding().padding(bottom = Spacing.md)) {
                Box(Modifier.fillMaxWidth().padding(top = Spacing.sm), contentAlignment = Alignment.Center) {
                    Box(Modifier.size(width = 36.dp, height = 4.dp).clip(CircleShape).background(colors.onSurfaceVariant.copy(alpha = 0.4f)))
                }
                val pack = browsing
                when {
                    pack != null -> PackIcons(pack, onBack = { browsing = null }) { name ->
                        state.draft = state.draft.copy(packPackage = pack.packageName, drawable = name, file = null)
                        browsing = null
                    }
                    choosingPack -> PackList(state.packs, onBack = { choosingPack = false }) {
                        choosingPack = false
                        browsing = it
                    }
                    else -> MainPage(
                        state, label, { label = it },
                        onPack = { if (state.packs.size == 1) browsing = state.packs[0] else choosingPack = true },
                        actions = actions,
                        onSave = {
                            actions.saveEditApp(state.app, state.draft.copy(label = label.trim().ifEmpty { null }))
                        },
                    )
                }
            }
        }
    }
}

@Composable
private fun MainPage(
    state: EditAppState,
    label: String,
    onLabel: (String) -> Unit,
    onPack: () -> Unit,
    actions: OverlayActions,
    onSave: () -> Unit,
) {
    val colors = MaterialTheme.colorScheme
    val context = LocalContext.current
    val densityDpi = LocalConfiguration.current.densityDpi
    val draft = state.draft
    // The icon as it will look: the chosen one, else the current one.
    val preview by produceState<ImageBitmap?>(state.app.image, draft) {
        value = withContext(Dispatchers.IO) {
            when {
                draft.file != null -> android.graphics.BitmapFactory.decodeFile(File(IconFiles.dir(context), draft.file).path)?.asImageBitmap()
                draft.packPackage != null && draft.drawable != null ->
                    IconPack.load(context, draft.packPackage)?.drawableNamed(draft.drawable, densityDpi)
                        ?.toBitmap(PREVIEW_PX, PREVIEW_PX)?.asImageBitmap()
                else -> null
            }
        } ?: state.app.image
    }
    Column(Modifier.padding(horizontal = Spacing.lg), verticalArrangement = Arrangement.spacedBy(Spacing.md)) {
        Text(stringResource(R.string.edit_app_title), style = MaterialTheme.typography.headlineMedium, color = colors.onSurface, modifier = Modifier.padding(top = Spacing.sm))
        Row(verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.spacedBy(Spacing.md)) {
            preview?.let { Image(it, contentDescription = null, modifier = Modifier.size(64.dp)) }
            val hint = state.app.label
            Surface(shape = CircleShape, color = colors.surfaceContainerHighest, modifier = Modifier.weight(1f).height(48.dp).oneDepthBorder()) {
                Box(Modifier.padding(horizontal = Spacing.mdLg), contentAlignment = Alignment.CenterStart) {
                    val desc = stringResource(R.string.edit_app_label)
                    BasicTextField(
                        value = label,
                        onValueChange = onLabel,
                        singleLine = true,
                        textStyle = MaterialTheme.typography.bodyLarge.copy(color = colors.onSurface),
                        cursorBrush = SolidColor(colors.primary),
                        modifier = Modifier.fillMaxWidth().semantics { contentDescription = desc },
                        decorationBox = { inner ->
                            if (label.isEmpty()) Text(hint, style = MaterialTheme.typography.bodyLarge, color = colors.onSurfaceVariant, maxLines = 1)
                            inner()
                        },
                    )
                }
            }
        }
        Row(horizontalArrangement = Arrangement.spacedBy(Spacing.sm)) {
            if (state.packs.isNotEmpty()) SecondaryButton(stringResource(R.string.edit_app_from_pack), onPack, icon = R.drawable.ph_palette)
            SecondaryButton(stringResource(R.string.edit_app_image), actions::pickIconImage, icon = R.drawable.ph_wallpaper)
        }
        if (draft.hasIcon) {
            SecondaryButton(stringResource(R.string.edit_app_default_icon), { state.draft = draft.copy(packPackage = null, drawable = null, file = null) }, icon = R.drawable.ph_close)
        }
        if (state.showCategory) {
            SectionHeader(stringResource(R.string.edit_app_category), Modifier.padding(top = Spacing.sm))
            val auto = state.app.autoCategory ?: AppCategories.Category.OTHERS
            val autoLabel = stringResource(R.string.edit_app_category_auto, stringResource(auto.titleRes()))
            val chosen = AppCategories.Category.fromKey(draft.category)
            CategoryDropdown(
                current = chosen?.let { stringResource(it.titleRes()) } ?: autoLabel,
                autoLabel = autoLabel,
                selected = chosen,
                onPick = { state.draft = draft.copy(category = it?.key) },
            )
        }
        Row(horizontalArrangement = Arrangement.spacedBy(Spacing.sm), modifier = Modifier.padding(top = Spacing.sm)) {
            Spacer(Modifier.weight(1f))
            SecondaryButton(stringResource(R.string.cancel), actions::closeEditApp)
            PrimaryButton(stringResource(R.string.save), onSave)
        }
    }
}

/**
 * A pill showing the current category; tapping it opens the list right below, at most
 * [DROPDOWN_MAX] tall and scrollable, in OneDepth's list style.
 */
@Composable
private fun CategoryDropdown(current: String, autoLabel: String, selected: AppCategories.Category?, onPick: (AppCategories.Category?) -> Unit) {
    val colors = MaterialTheme.colorScheme
    var open by remember { mutableStateOf(false) }
    Column(verticalArrangement = Arrangement.spacedBy(Spacing.xs)) {
        Surface(
            onClick = { open = !open },
            shape = CircleShape,
            color = colors.surfaceContainerHighest,
            modifier = Modifier.fillMaxWidth().height(44.dp).oneDepthBorder(),
        ) {
            Row(Modifier.padding(horizontal = Spacing.md), verticalAlignment = Alignment.CenterVertically) {
                Text(current, style = MaterialTheme.typography.bodyLarge, color = colors.onSurface, modifier = Modifier.weight(1f), maxLines = 1)
                Icon(painterResource(R.drawable.ph_caret_down), contentDescription = null, tint = colors.onSurfaceVariant, modifier = Modifier.size(18.dp).rotate(if (open) 180f else 0f))
            }
        }
        if (open) {
            Surface(shape = RoundedCornerShape(24.dp), color = colors.surfaceContainerHighest, modifier = Modifier.fillMaxWidth().oneDepthBorder(cornerRadius = 24.dp)) {
                LazyColumn(Modifier.heightIn(max = DROPDOWN_MAX).padding(Spacing.xsSm), verticalArrangement = Arrangement.spacedBy(Spacing.xxs)) {
                    item { DropdownRow(autoLabel, selected == null) { onPick(null); open = false } }
                    items(AppCategories.Category.entries.toList(), key = { it.key }) { c ->
                        DropdownRow(stringResource(c.titleRes()), selected == c) { onPick(c); open = false }
                    }
                }
            }
        }
    }
}

@Composable
private fun DropdownRow(label: String, selected: Boolean, onClick: () -> Unit) {
    val colors = MaterialTheme.colorScheme
    Row(
        Modifier
            .fillMaxWidth()
            .clip(CircleShape)
            .background(if (selected) colors.secondaryContainer else androidx.compose.ui.graphics.Color.Transparent)
            .clickable(role = Role.RadioButton, onClick = onClick)
            .padding(horizontal = Spacing.md, vertical = Spacing.sm),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Text(label, style = MaterialTheme.typography.bodyLarge, color = if (selected) colors.onSecondaryContainer else colors.onSurface, modifier = Modifier.weight(1f))
        if (selected) Icon(painterResource(R.drawable.ph_check), contentDescription = null, tint = colors.onSecondaryContainer, modifier = Modifier.size(18.dp))
    }
}

private val DROPDOWN_MAX = 220.dp

@Composable
private fun Header(title: String, onBack: () -> Unit) {
    Row(Modifier.padding(start = Spacing.sm, end = Spacing.lg, top = Spacing.xs), verticalAlignment = Alignment.CenterVertically) {
        IconButton(onClick = onBack) {
            Icon(painterResource(R.drawable.ph_caret_left), contentDescription = stringResource(R.string.back), modifier = Modifier.size(22.dp))
        }
        Text(title, style = MaterialTheme.typography.headlineSmall, color = MaterialTheme.colorScheme.onSurface, maxLines = 1, overflow = TextOverflow.Ellipsis)
    }
}

@Composable
private fun PackList(packs: List<IconPackInfo>, onBack: () -> Unit, onPick: (IconPackInfo) -> Unit) {
    Header(stringResource(R.string.edit_app_choose_pack), onBack)
    LazyColumn(contentPadding = PaddingValues(horizontal = Spacing.sm, vertical = Spacing.sm)) {
        items(packs, key = { it.packageName }) { p ->
            Row(
                Modifier
                    .fillMaxWidth()
                    .clip(CircleShape)
                    .clickable(role = Role.Button) { onPick(p) }
                    .padding(horizontal = Spacing.mdLg, vertical = Spacing.smMd),
            ) {
                Text(p.label, style = MaterialTheme.typography.bodyLarge, color = MaterialTheme.colorScheme.onSurface)
            }
        }
    }
}

/** Every icon of [pack], searchable by name. */
@Composable
private fun PackIcons(pack: IconPackInfo, onBack: () -> Unit, onPick: (String) -> Unit) {
    val context = LocalContext.current
    val densityDpi = LocalConfiguration.current.densityDpi
    val colors = MaterialTheme.colorScheme
    var query by remember { mutableStateOf("") }
    val loaded by produceState<Pair<IconPack?, List<String>>?>(null, pack) {
        value = withContext(Dispatchers.IO) {
            val p = IconPack.load(context, pack.packageName)
            p to (p?.allDrawables(context) ?: emptyList())
        }
    }
    Header(pack.label, onBack)
    Surface(
        shape = CircleShape,
        color = colors.surfaceContainerHighest,
        modifier = Modifier.padding(horizontal = Spacing.md, vertical = Spacing.sm).fillMaxWidth().height(48.dp).oneDepthBorder(),
    ) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(painterResource(R.drawable.ph_search), contentDescription = null, tint = colors.onSurfaceVariant, modifier = Modifier.padding(start = Spacing.md, end = Spacing.smMd).size(20.dp))
            val desc = stringResource(R.string.edit_app_search_icons)
            BasicTextField(
                value = query,
                onValueChange = { query = it },
                singleLine = true,
                textStyle = MaterialTheme.typography.bodyLarge.copy(color = colors.onSurface),
                cursorBrush = SolidColor(colors.primary),
                modifier = Modifier.weight(1f).padding(end = Spacing.md).semantics { contentDescription = desc },
                decorationBox = { inner ->
                    if (query.isEmpty()) Text(desc, style = MaterialTheme.typography.bodyLarge, color = colors.onSurfaceVariant, maxLines = 1)
                    inner()
                },
            )
        }
    }
    val (iconPack, names) = loaded ?: (null to emptyList())
    val shown = remember(names, query) {
        val q = query.trim().replace(' ', '_')
        if (q.isEmpty()) names else names.filter { it.contains(q, ignoreCase = true) }
    }
    if (iconPack == null) return
    LazyVerticalGrid(
        columns = GridCells.Adaptive(72.dp),
        contentPadding = PaddingValues(horizontal = Spacing.md, vertical = Spacing.sm),
        modifier = Modifier.fillMaxWidth(),
    ) {
        items(shown, key = { it }) { name ->
            val img by produceState<ImageBitmap?>(null, name) {
                value = withContext(Dispatchers.IO) {
                    try {
                        iconPack.drawableNamed(name, densityDpi)?.toBitmap(PREVIEW_PX / 2, PREVIEW_PX / 2)?.asImageBitmap()
                    } catch (e: RuntimeException) {
                        null
                    }
                }
            }
            Box(
                Modifier
                    .aspectRatio(1f)
                    .clip(RoundedCornerShape(16.dp))
                    .clickable(role = Role.Button) { onPick(name) }
                    .padding(Spacing.sm)
                    .semantics { contentDescription = name.replace('_', ' ') },
                contentAlignment = Alignment.Center,
            ) {
                img?.let { Image(it, contentDescription = null, modifier = Modifier.fillMaxSize()) }
            }
        }
    }
}
