package io.github.jacomartins.canyonui.ui.overlay

import android.appwidget.AppWidgetHostView
import android.appwidget.AppWidgetProviderInfo
import android.graphics.Rect
import android.os.Build
import android.view.View
import android.widget.RemoteViews
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.foundation.ExperimentalFoundationApi
import androidx.compose.foundation.Image
import androidx.compose.foundation.background
import androidx.compose.foundation.clickable
import androidx.compose.foundation.combinedClickable
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.fillMaxHeight
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.imePadding
import androidx.compose.foundation.layout.navigationBars
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.layout.windowInsetsPadding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
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
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.layout.layout
import androidx.compose.ui.draw.clipToBounds
import androidx.compose.ui.draw.clip
import androidx.compose.ui.draw.rotate
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.SolidColor
import androidx.compose.ui.hapticfeedback.HapticFeedbackType
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.layout.boundsInWindow
import androidx.compose.ui.layout.onGloballyPositioned
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalHapticFeedback
import androidx.compose.ui.res.painterResource
import androidx.compose.ui.res.pluralStringResource
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.dp
import androidx.compose.ui.viewinterop.AndroidView
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.ui.design.Spacing
import io.github.jacomartins.canyonui.ui.design.backdrop
import io.github.jacomartins.canyonui.ui.design.backdropColor
import io.github.jacomartins.canyonui.ui.design.oneDepthBorder
import io.github.jacomartins.canyonui.ui.widget.PickerGroup
import io.github.jacomartins.canyonui.ui.widget.PickerWidget
import io.github.jacomartins.canyonui.ui.widget.WidgetCatalog
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlin.math.roundToInt

/**
 * What the widget picker shows. With [stackSpan] set it adds to a stack, so only widgets that
 * can be that size are offered.
 */
class WidgetPickerState(val groups: List<PickerGroup>, val stackSpan: Pair<Int, Int>? = null)

private val SheetShape = RoundedCornerShape(topStart = 32.dp, topEnd = 32.dp)

/** The widget sheet's colour over the Transparency effect (the folder panel is 0.72). */
private const val SHEET_ALPHA = 0.85f
private const val CARD_WIDTH_DP = 148

/** Bottom sheet listing widgets by app, with search. Tap adds to the current page; long-press drags. */
@Composable
fun WidgetPicker(picker: WidgetPickerState, actions: OverlayActions) {
    var query by remember { mutableStateOf("") }
    var expanded by remember { mutableStateOf<PickerGroup?>(null) }
    val groups = remember(picker, query) { filter(picker, query.trim()) }
    val colors = MaterialTheme.colorScheme
    val title = stringResource(if (picker.stackSpan != null) R.string.widgets_for_stack else R.string.widgets)

    Box(
        Modifier
            .fillMaxSize()
            .background(colors.scrim.copy(alpha = 0.4f))
            .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null, onClick = actions::closePicker),
    ) {
        Surface(
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .fillMaxWidth()
                .fillMaxHeight(0.9f)
                .oneDepthBorder(cornerRadius = 32.dp)
                // Translucent, frosted or glass by the Transparency setting, like the other panels.
                .backdrop(SheetShape)
                // Taps on the sheet itself do not reach the scrim.
                .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null) {},
            shape = SheetShape,
            color = backdropColor(colors.surfaceContainer.copy(alpha = SHEET_ALPHA)),
        ) {
            Column(Modifier.windowInsetsPadding(WindowInsets.navigationBars).imePadding()) {
                Box(Modifier.fillMaxWidth().padding(top = Spacing.sm), contentAlignment = Alignment.Center) {
                    Box(Modifier.size(width = 36.dp, height = 4.dp).clip(CircleShape).background(colors.onSurfaceVariant.copy(alpha = 0.4f)))
                }
                Row(Modifier.padding(start = Spacing.lg, end = Spacing.sm, top = Spacing.sm), verticalAlignment = Alignment.CenterVertically) {
                    Text(title, style = MaterialTheme.typography.headlineMedium, color = colors.onSurface, modifier = Modifier.weight(1f))
                    IconButton(onClick = actions::closePicker) {
                        Icon(painterResource(R.drawable.ph_close), contentDescription = stringResource(R.string.close), tint = colors.onSurfaceVariant, modifier = Modifier.size(22.dp))
                    }
                }
                SearchField(query, { query = it }, Modifier.padding(horizontal = Spacing.md, vertical = Spacing.sm))
                if (groups.isEmpty()) {
                    Text(
                        stringResource(R.string.widgets_none),
                        style = MaterialTheme.typography.bodyLarge,
                        color = colors.onSurfaceVariant,
                        modifier = Modifier.padding(Spacing.lg),
                    )
                }
                LazyColumn(contentPadding = PaddingValues(bottom = Spacing.lg), modifier = Modifier.weight(1f)) {
                    items(groups, key = { "${it.packageName}#${it.user.hashCode()}" }) { g ->
                        val open = query.isNotBlank() || groups.size == 1 || expanded?.let { it.packageName == g.packageName && it.user == g.user } == true
                        GroupRow(g, open) { expanded = if (open) null else g }
                        if (open) WidgetRow(g, actions)
                    }
                }
            }
        }
    }
}

private fun filter(picker: WidgetPickerState, q: String): List<PickerGroup> {
    val span = picker.stackSpan
    return picker.groups.mapNotNull { g ->
        val fits = if (span == null) g.widgets else g.widgets.filter { w ->
            span.first in w.range.minX..w.range.maxX && span.second in w.range.minY..w.range.maxY
        }
        val hits = when {
            q.isEmpty() || g.label.contains(q, ignoreCase = true) -> fits
            else -> fits.filter { it.label.contains(q, ignoreCase = true) }
        }
        if (hits.isEmpty()) null else PickerGroup(g.packageName, g.user, g.label, g.icon, hits)
    }
}

@Composable
private fun SearchField(query: String, onChange: (String) -> Unit, modifier: Modifier) {
    val colors = MaterialTheme.colorScheme
    val label = stringResource(R.string.widgets_search)
    Surface(shape = CircleShape, color = colors.surfaceContainerHighest, modifier = modifier.fillMaxWidth().height(48.dp).oneDepthBorder()) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(painterResource(R.drawable.ph_search), contentDescription = null, tint = colors.onSurfaceVariant, modifier = Modifier.padding(start = Spacing.md, end = Spacing.smMd).size(20.dp))
            BasicTextField(
                value = query,
                onValueChange = onChange,
                singleLine = true,
                textStyle = MaterialTheme.typography.bodyLarge.copy(color = colors.onSurface),
                cursorBrush = SolidColor(colors.primary),
                modifier = Modifier.weight(1f).semantics { contentDescription = label },
                decorationBox = { inner ->
                    Box(contentAlignment = Alignment.CenterStart) {
                        if (query.isEmpty()) Text(label, style = MaterialTheme.typography.bodyLarge, color = colors.onSurfaceVariant, maxLines = 1)
                        inner()
                    }
                },
            )
            if (query.isNotEmpty()) {
                IconButton(onClick = { onChange("") }) {
                    Icon(painterResource(R.drawable.ph_close), contentDescription = stringResource(R.string.clear_search), tint = colors.onSurfaceVariant, modifier = Modifier.size(18.dp))
                }
            } else {
                Spacer(Modifier.width(Spacing.md))
            }
        }
    }
}

@Composable
private fun GroupRow(g: PickerGroup, open: Boolean, onToggle: () -> Unit) {
    val colors = MaterialTheme.colorScheme
    val turn by animateFloatAsState(if (open) 180f else 0f, label = "caret")
    Row(
        Modifier
            .fillMaxWidth()
            .clickable(onClick = onToggle)
            .padding(horizontal = Spacing.lg, vertical = Spacing.smMd),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        val icon = g.icon
        if (icon != null) Image(icon, contentDescription = null, modifier = Modifier.size(40.dp)) else Spacer(Modifier.size(40.dp))
        Column(Modifier.weight(1f).padding(start = Spacing.md)) {
            Text(g.label, style = MaterialTheme.typography.titleMedium, color = colors.onSurface, maxLines = 1, overflow = TextOverflow.Ellipsis)
            Text(
                pluralStringResource(R.plurals.widget_count, g.widgets.size, g.widgets.size),
                style = MaterialTheme.typography.bodySmall,
                color = colors.onSurfaceVariant,
            )
        }
        Icon(painterResource(R.drawable.ph_caret_down), contentDescription = null, tint = colors.onSurfaceVariant, modifier = Modifier.size(20.dp).rotate(turn))
    }
}

@Composable
private fun WidgetRow(g: PickerGroup, actions: OverlayActions) {
    LazyRow(
        contentPadding = PaddingValues(horizontal = Spacing.lg),
        horizontalArrangement = Arrangement.spacedBy(Spacing.md),
        modifier = Modifier.padding(bottom = Spacing.md),
    ) {
        items(g.widgets, key = { it.info.provider.flattenToString() }) { w -> WidgetCard(w, actions) }
    }
}

@OptIn(ExperimentalFoundationApi::class)
@Composable
private fun WidgetCard(w: PickerWidget, actions: OverlayActions) {
    val colors = MaterialTheme.colorScheme
    val context = LocalContext.current
    val haptics = LocalHapticFeedback.current
    val ratio = (w.spanY.toFloat() / w.spanX).coerceIn(0.5f, 1.4f)
    val image by produceState<ImageBitmap?>(null, w.info) {
        value = withContext(Dispatchers.IO) { WidgetCatalog.previewImage(context, w.info, 480) }
    }
    // Android 15+: a preview the widget generated from its real content (what many current
    // widgets ship instead of a preview layout or image).
    val generated by produceState<RemoteViews?>(null, w.info) {
        value = withContext(Dispatchers.IO) { WidgetCatalog.generatedPreview(context, w.info) }
    }
    val bounds = remember { arrayOfNulls<androidx.compose.ui.geometry.Rect>(1) }
    val sizeText = stringResource(R.string.widget_size, w.spanX, w.spanY)
    Column(Modifier.width(CARD_WIDTH_DP.dp)) {
        Box(
            Modifier
                .fillMaxWidth()
                .height((CARD_WIDTH_DP * ratio).dp)
                .onGloballyPositioned { bounds[0] = it.boundsInWindow() }
                .clip(RoundedCornerShape(20.dp))
                .background(colors.surfaceContainerHighest)
                .oneDepthBorder(cornerRadius = 20.dp),
            contentAlignment = Alignment.Center,
        ) {
            val img = image
            Box(Modifier.fillMaxSize().padding(Spacing.sm), contentAlignment = Alignment.Center) {
                when {
                    generated != null && Build.VERSION.SDK_INT >= 31 -> RemotePreview(w.info, w.spanX, w.spanY) { generated!! }
                    !WidgetCatalog.hasPreviewImage(w.info) && Build.VERSION.SDK_INT >= 31 && w.info.previewLayout != 0 ->
                        RemotePreview(w.info, w.spanX, w.spanY) { RemoteViews(w.info.provider.packageName, w.info.previewLayout) }
                    img != null -> Image(img, contentDescription = null, contentScale = ContentScale.Fit, modifier = Modifier.fillMaxSize())
                }
            }
            // On top of the preview, so a layout preview's own views never take the touch.
            Box(
                Modifier
                    .matchParentSize()
                    .combinedClickable(
                        onClickLabel = stringResource(R.string.widget_add),
                        onClick = { actions.addWidget(w) },
                        onLongClick = {
                            haptics.performHapticFeedback(HapticFeedbackType.LongPress)
                            bounds[0]?.let { b ->
                                actions.armWidgetDrag(w, image, Rect(b.left.roundToInt(), b.top.roundToInt(), b.right.roundToInt(), b.bottom.roundToInt()))
                            }
                        },
                    )
                    .semantics { contentDescription = "${w.label}, $sizeText" },
            )
        }
        Text(
            w.label,
            style = MaterialTheme.typography.titleSmall,
            color = colors.onSurface,
            maxLines = 2,
            overflow = TextOverflow.Ellipsis,
            modifier = Modifier.padding(top = Spacing.sm),
        )
        Text(sizeText, style = MaterialTheme.typography.labelMedium, color = colors.onSurfaceVariant)
    }
}

/**
 * Android 12+ widgets can describe their preview as a layout (or, on Android 15+, a generated
 * preview) instead of an image. The layout is
 * laid out at the widget's real size on home ([spanX] x [spanY] cells) and scaled down into the
 * card: wide widgets (four cells) built for that width come out empty or broken when squeezed
 * into the card's own small size.
 */
@androidx.annotation.RequiresApi(31)
@Composable
private fun RemotePreview(info: AppWidgetProviderInfo, spanX: Int, spanY: Int, views: () -> RemoteViews) {
    // A home cell is about a quarter of the screen width, and a little taller than wide.
    val cellW = androidx.compose.ui.platform.LocalWindowInfo.current.containerSize.width / 4
    val realW = cellW * spanX
    val realH = (cellW * 1.15f).toInt() * spanY
    AndroidView(
        factory = { c ->
            try {
                AppWidgetHostView(c).apply {
                    setAppWidget(-1, info)
                    updateAppWidget(views())
                    // A preview: no touches reach the provider's views.
                    importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_NO_HIDE_DESCENDANTS
                }
            } catch (e: RuntimeException) {
                View(c)
            }
        },
        modifier = Modifier
            .fillMaxSize()
            .clipToBounds()
            .layout { measurable, constraints ->
                val p = measurable.measure(androidx.compose.ui.unit.Constraints.fixed(realW, realH))
                val scale = minOf(constraints.maxWidth / realW.toFloat(), constraints.maxHeight / realH.toFloat())
                layout(constraints.maxWidth, constraints.maxHeight) {
                    val x = ((constraints.maxWidth - realW * scale) / 2f).roundToInt()
                    val y = ((constraints.maxHeight - realH * scale) / 2f).roundToInt()
                    p.placeWithLayer(x, y) {
                        scaleX = scale
                        scaleY = scale
                        transformOrigin = androidx.compose.ui.graphics.TransformOrigin(0f, 0f)
                    }
                }
            },
    )
}
