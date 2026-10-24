package io.github.jacomartins.canyonui.ui.design

import android.graphics.Rect
import androidx.annotation.DrawableRes
import androidx.compose.foundation.clickable
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.IntrinsicSize
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.layout.widthIn
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.draw.clip
import androidx.compose.ui.draw.shadow
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.layout.layout
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.res.painterResource
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.unit.dp

/** One row of a [MenuLayer]: an icon (Phosphor Regular), a label and what it does. */
class MenuEntry(@DrawableRes val icon: Int, val label: String, val action: () -> Unit)

/**
 * The long-press menu used everywhere (home, folders, the app drawer, widgets): a panel next to
 * [anchor] (window coordinates), above it when there is room, else below. A tap outside closes it.
 */
@Composable
fun MenuLayer(
    anchor: Rect,
    entries: List<MenuEntry>,
    onDismiss: () -> Unit,
    modifier: Modifier = Modifier,
    /** 0 = hidden, 1 = open: the panel fades in and out where it stands. */
    progress: (() -> Float)? = null,
) {
    val density = LocalDensity.current
    // Close to the item it belongs to.
    val margin = with(density) { 6.dp.roundToPx() }
    Box(
        Modifier
            .fillMaxSize()
            .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null, onClick = onDismiss),
    ) {
        Surface(
            modifier = Modifier
                .layout { measurable, constraints ->
                    val p = measurable.measure(constraints.copy(minWidth = 0, minHeight = 0))
                    layout(constraints.maxWidth, constraints.maxHeight) {
                        val a = anchor
                        val x = (a.centerX() - p.width / 2).coerceIn(margin, constraints.maxWidth - p.width - margin)
                        // Above the icon when there is room, otherwise below it.
                        val above = a.top - p.height - margin
                        val y = if (above > margin * 8) above else (a.bottom + margin).coerceAtMost(constraints.maxHeight - p.height - margin)
                        p.place(x, y)
                    }
                }
                .graphicsLayer { alpha = progress?.invoke() ?: 1f }
                .then(modifier)
                .shadow(Spacing.sm, MenuShape, ambientColor = Color.Black.copy(alpha = 0.25f), spotColor = Color.Black.copy(alpha = 0.375f))
                .oneDepthBorder(cornerRadius = MENU_RADIUS)
                .backdrop(MenuShape),
            shape = MenuShape,
            color = backdropColor(MaterialTheme.colorScheme.surfaceContainer.copy(alpha = 0.97f)),
        ) {
            // OneDepth list, compact: an inset column of pill rows (16 x 8 padding, 12 between
            // icon and text, 2 between rows). The panel's radius is the row's plus the inset, so a
            // pressed row sits concentric with the panel.
            Column(
                Modifier.width(IntrinsicSize.Max).padding(Spacing.xsSm),
                verticalArrangement = Arrangement.spacedBy(Spacing.xxs),
            ) {
                for (e in entries) {
                    Row(
                        Modifier
                            .fillMaxWidth()
                            .widthIn(min = 184.dp)
                            .clip(CircleShape)
                            .clickable(role = Role.Button) { onDismiss(); e.action() }
                            .padding(horizontal = Spacing.md, vertical = Spacing.sm),
                        verticalAlignment = Alignment.CenterVertically,
                        horizontalArrangement = Arrangement.spacedBy(Spacing.smMd),
                    ) {
                        Icon(painterResource(e.icon), contentDescription = null, modifier = Modifier.size(20.dp), tint = MaterialTheme.colorScheme.onSurface)
                        Text(e.label, style = MaterialTheme.typography.bodyLarge, color = MaterialTheme.colorScheme.onSurface)
                    }
                }
            }
        }
    }
}

/** A row is about 40dp tall (pill radius 20) inside a 6dp inset. */
private val MENU_RADIUS = 26.dp
private val MenuShape = RoundedCornerShape(MENU_RADIUS)
