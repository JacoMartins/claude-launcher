package io.github.jacomartins.canyonui.ui.design

import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.core.tween
import androidx.compose.animation.fadeIn
import androidx.compose.animation.slideInVertically
import androidx.compose.foundation.clickable
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.heightIn
import androidx.compose.foundation.layout.ime
import androidx.compose.foundation.layout.navigationBars
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.union
import androidx.compose.foundation.layout.widthIn
import androidx.compose.foundation.layout.windowInsetsPadding
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.ProvideTextStyle
import androidx.compose.material3.Surface
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.ui.window.Dialog
import androidx.compose.ui.window.DialogProperties

/**
 * A question as a sheet at the bottom of the screen ("Set wallpaper on", "Add from", pickers),
 * near the thumb, with the interface's border. Same slots as Material's AlertDialog so either can
 * be used; confirmations ("Delete this page?") stay centred AlertDialogs.
 */
@Composable
fun SheetDialog(
    onDismissRequest: () -> Unit,
    confirmButton: @Composable () -> Unit,
    modifier: Modifier = Modifier,
    dismissButton: (@Composable () -> Unit)? = null,
    title: (@Composable () -> Unit)? = null,
    text: (@Composable () -> Unit)? = null,
) {
    Dialog(
        onDismissRequest = onDismissRequest,
        properties = DialogProperties(usePlatformDefaultWidth = false, decorFitsSystemWindows = false),
    ) {
        var shown by remember { mutableStateOf(false) }
        LaunchedEffect(Unit) { shown = true }
        val density = androidx.compose.ui.platform.LocalDensity.current
        val maxHeight = with(density) { (androidx.compose.ui.platform.LocalWindowInfo.current.containerSize.height * 0.85f).toDp() }
        Box(
            Modifier
                .fillMaxSize()
                // A tap above the sheet closes it, as outside a dialog.
                .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null, onClick = onDismissRequest)
                .windowInsetsPadding(WindowInsets.navigationBars.union(WindowInsets.ime)),
            contentAlignment = Alignment.BottomCenter,
        ) {
            AnimatedVisibility(
                visible = shown,
                enter = slideInVertically(tween(SHEET_MS)) { it / 3 } + fadeIn(tween(SHEET_MS)),
            ) {
                Surface(
                    shape = SheetShape,
                    color = MaterialTheme.colorScheme.surfaceContainerHigh,
                    modifier = modifier
                        .padding(horizontal = Spacing.sm, vertical = Spacing.sm)
                        .widthIn(max = 560.dp)
                        .fillMaxWidth()
                        .heightIn(max = maxHeight)
                        // Taps inside stay inside.
                        .clickable(interactionSource = remember { MutableInteractionSource() }, indication = null) { }
                        .oneDepthBorder(cornerRadius = SHEET_RADIUS),
                ) {
                    Column(Modifier.padding(Spacing.lg), verticalArrangement = Arrangement.spacedBy(Spacing.md)) {
                        title?.let { ProvideTextStyle(MaterialTheme.typography.headlineSmall) { it() } }
                        text?.let {
                            Box(Modifier.weight(1f, fill = false)) {
                                ProvideTextStyle(MaterialTheme.typography.bodyMedium) { it() }
                            }
                        }
                        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.spacedBy(Spacing.sm, Alignment.End)) {
                            dismissButton?.invoke()
                            confirmButton()
                        }
                    }
                }
            }
        }
    }
}

private val SHEET_RADIUS = 28.dp
private val SheetShape = RoundedCornerShape(SHEET_RADIUS)
private const val SHEET_MS = 220
