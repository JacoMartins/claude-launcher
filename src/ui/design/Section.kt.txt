package io.github.jacomartins.canyonui.ui.design

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp

/**
 * The title above a group of settings or choices: 14sp bold in the text colour (white in dark
 * mode), sitting on the page above the group's container rather than inside it.
 */
@Composable
fun SectionHeader(text: String, modifier: Modifier = Modifier) {
    Text(
        text,
        style = MaterialTheme.typography.labelLarge.copy(fontSize = 14.sp),
        color = MaterialTheme.colorScheme.onSurface,
        modifier = modifier,
    )
}

/** The page behind settings-style sections: black in dark mode; light grey in light mode, under white containers. */
@Composable
fun settingsPageColor() = if (isDarkSurface()) MaterialTheme.colorScheme.background else MaterialTheme.colorScheme.surfaceContainer

/**
 * A settings-style section: an optional [title] straight on the page ([SectionHeader]), then its
 * content in a rounded container a step lighter than the page.
 */
@Composable
fun SettingsSection(title: String? = null, onClick: (() -> Unit)? = null, content: @Composable () -> Unit) {
    Column(Modifier.fillMaxWidth().padding(top = if (title != null) Spacing.md else Spacing.none)) {
        if (title != null) SectionHeader(title, Modifier.padding(start = Spacing.mdLg, bottom = Spacing.sm))
        Surface(
            shape = RoundedCornerShape(24.dp),
            color = if (isDarkSurface()) MaterialTheme.colorScheme.surfaceContainer else MaterialTheme.colorScheme.surfaceContainerLowest,
            modifier = Modifier.fillMaxWidth(),
        ) {
            Column(
                (if (onClick != null) Modifier.clickable(onClick = onClick) else Modifier).padding(horizontal = Spacing.mdLg, vertical = Spacing.md),
                verticalArrangement = Arrangement.spacedBy(Spacing.sm),
            ) { content() }
        }
    }
}
