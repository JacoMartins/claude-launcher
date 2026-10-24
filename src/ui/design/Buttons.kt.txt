package io.github.jacomartins.canyonui.ui.design

import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.text.style.TextAlign
import androidx.annotation.DrawableRes
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.RowScope
import androidx.compose.foundation.layout.widthIn
import androidx.compose.material3.ProvideTextStyle
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.heightIn
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.material3.LocalMinimumInteractiveComponentSize
import androidx.compose.runtime.Composable
import androidx.compose.runtime.CompositionLocalProvider
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.res.painterResource
import androidx.compose.ui.semantics.selected
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.unit.dp

/** Which of the two button looks: blue [Primary] for the main action, grey [Secondary] for the rest. */
enum class ButtonKind { Primary, Secondary }

/**
 * The one button of the interface: a pill (or a circle when it only holds an icon) with the depth
 * ring and bold content. [ButtonKind.Primary] is filled with the primary blue,
 * [ButtonKind.Secondary] with the neutral grey; any content can go inside.
 */
@Composable
fun PillButton(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    kind: ButtonKind = ButtonKind.Primary,
    enabled: Boolean = true,
    contentPadding: PaddingValues = PaddingValues(horizontal = Spacing.mdLg, vertical = Spacing.sm),
    content: @Composable RowScope.() -> Unit,
) {
    val colors = MaterialTheme.colorScheme
    val (container, onContainer) = when (kind) {
        ButtonKind.Primary -> colors.primary to colors.onPrimary
        ButtonKind.Secondary -> colors.secondaryContainer to colors.onSecondaryContainer
    }
    // Material grows clickable surfaces to a 48dp touch target, which made the ring trace a larger
    // box than the visible pill (a doubled outline). Turn that off and give the pill a 44dp
    // minimum size instead, so the ring, the fill and the touch area are the same shape.
    CompositionLocalProvider(LocalMinimumInteractiveComponentSize provides Dp.Unspecified) {
        Surface(
            onClick = onClick,
            enabled = enabled,
            shape = CircleShape,
            color = if (enabled) container else container.copy(alpha = 0.38f),
            contentColor = if (enabled) onContainer else onContainer.copy(alpha = 0.6f),
            modifier = modifier.heightIn(min = 44.dp).widthIn(min = 44.dp).oneDepthBorder(),
        ) {
            // Labels centred everywhere, also when a long one fills a wide button.
            ProvideTextStyle(MaterialTheme.typography.labelLarge.copy(textAlign = TextAlign.Center)) {
                Row(
                    Modifier.padding(contentPadding),
                    horizontalArrangement = Arrangement.spacedBy(Spacing.xsSm, Alignment.CenterHorizontally),
                    verticalAlignment = Alignment.CenterVertically,
                    content = content,
                )
            }
        }
    }
}

/** A [PillButton] with a label and an optional trailing icon. */
@Composable
fun PillButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    kind: ButtonKind = ButtonKind.Primary,
    @DrawableRes icon: Int? = null,
    enabled: Boolean = true,
) = PillButton(onClick, modifier, kind, enabled) {
    Text(text, maxLines = 1, overflow = TextOverflow.Ellipsis, textAlign = TextAlign.Center, modifier = Modifier.weight(1f, fill = false))
    if (icon != null) Icon(painterResource(icon), contentDescription = null, modifier = Modifier.size(18.dp))
}

/** A round [PillButton] holding only an icon; [label] is read out by accessibility services. */
@Composable
fun IconPillButton(
    @DrawableRes icon: Int,
    label: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    kind: ButtonKind = ButtonKind.Secondary,
    enabled: Boolean = true,
    size: Dp = 44.dp,
) = PillButton(
    onClick,
    modifier.size(size).semantics { contentDescription = label },
    kind, enabled,
    contentPadding = PaddingValues(0.dp),
) {
    Icon(painterResource(icon), contentDescription = null, modifier = Modifier.size(22.dp))
}

/** The main action: blue pill. */
@Composable
fun PrimaryButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    @DrawableRes icon: Int? = null,
    enabled: Boolean = true,
) = PillButton(text, onClick, modifier, ButtonKind.Primary, icon, enabled)

/** Every other action: grey pill, same shape, ring and label weight. */
@Composable
fun SecondaryButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    @DrawableRes icon: Int? = null,
    enabled: Boolean = true,
) = PillButton(text, onClick, modifier, ButtonKind.Secondary, icon, enabled)

/** A pill that is Primary when [selected] and Secondary otherwise (segmented choices). */
@Composable
fun ChoiceButton(text: String, selected: Boolean, onClick: () -> Unit, modifier: Modifier = Modifier) {
    val m = modifier.semantics { this.selected = selected }
    if (selected) PrimaryButton(text, onClick, m) else SecondaryButton(text, onClick, m)
}
