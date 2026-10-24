package io.github.jacomartins.canyonui.ui.settings

import android.content.Context
import android.content.pm.LauncherApps
import androidx.compose.foundation.Image
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.selection.selectable
import androidx.compose.foundation.selection.selectableGroup
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.RadioButton
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.produceState
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.unit.dp
import androidx.core.graphics.drawable.toBitmap
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.data.AppInfo
import io.github.jacomartins.canyonui.data.IconPack
import io.github.jacomartins.canyonui.data.IconPackInfo
import io.github.jacomartins.canyonui.ui.design.PrimaryButton
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

private const val PREVIEW_COUNT = 4
private const val PREVIEW_PX = 96

/**
 * Icon pack chooser: "Default (system icons)" first, then every installed pack, each with a
 * preview of a few of the user's own apps as that pack draws them.
 */
@Composable
fun IconPackPicker(
    selected: String?,
    apps: List<AppInfo>,
    onSelect: (String?) -> Unit,
    onDismiss: () -> Unit,
) {
    val context = LocalContext.current
    val packs by produceState<List<IconPackInfo>?>(null) {
        value = withContext(Dispatchers.Default) { IconPack.installed(context) }
    }
    io.github.jacomartins.canyonui.ui.design.SheetDialog(
        onDismissRequest = onDismiss,
        confirmButton = { PrimaryButton(stringResource(R.string.done), onDismiss) },
        title = { Text(stringResource(R.string.icon_pack)) },
        text = {
            val list = packs
            if (list == null) {
                Text(stringResource(R.string.loading))
            } else {
                LazyColumn(Modifier.selectableGroup()) {
                    item(key = "default") {
                        PackRow(stringResource(R.string.icon_pack_default), null, selected == null, apps) { onSelect(null) }
                    }
                    items(list, key = { it.packageName }) { pack ->
                        PackRow(pack.label, pack.packageName, selected == pack.packageName, apps) { onSelect(pack.packageName) }
                    }
                    if (list.isEmpty()) {
                        item(key = "none") {
                            Text(
                                stringResource(R.string.icon_pack_none_installed),
                                style = MaterialTheme.typography.bodySmall,
                                color = MaterialTheme.colorScheme.onSurfaceVariant,
                                modifier = Modifier.padding(top = 8.dp),
                            )
                        }
                    }
                }
            }
        },
    )
}

@Composable
private fun PackRow(label: String, packageName: String?, selected: Boolean, apps: List<AppInfo>, onClick: () -> Unit) {
    val context = LocalContext.current
    val preview by produceState<List<ImageBitmap>>(emptyList(), packageName, apps) {
        value = withContext(Dispatchers.Default) { previewIcons(context, packageName, apps) }
    }
    Row(
        Modifier
            .fillMaxWidth()
            .selectable(selected = selected, role = Role.RadioButton, onClick = onClick)
            .padding(vertical = 8.dp),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        RadioButton(selected = selected, onClick = null)
        Column(Modifier.padding(start = 12.dp)) {
            Text(label, style = MaterialTheme.typography.bodyLarge)
            Spacer(Modifier.height(6.dp))
            Row(horizontalArrangement = Arrangement.spacedBy(6.dp)) {
                for (img in preview) Image(img, contentDescription = null, modifier = Modifier.size(32.dp))
                if (preview.isEmpty()) Spacer(Modifier.height(32.dp))
            }
        }
    }
}

/** Up to four of the user's apps as [packageName] draws them (system icons when null). */
private fun previewIcons(context: Context, packageName: String?, apps: List<AppInfo>): List<ImageBitmap> {
    val density = context.resources.displayMetrics.densityDpi
    val mainApps = apps.filter { it.isMainProfile }
    if (packageName == null) {
        val la = context.getSystemService(LauncherApps::class.java)
        return mainApps.asSequence().take(PREVIEW_COUNT).mapNotNull { app ->
            la.resolveActivity(android.content.Intent().setComponent(app.component), app.user)
                ?.getIcon(density)?.toBitmap(PREVIEW_PX, PREVIEW_PX)?.asImageBitmap()
        }.toList()
    }
    val pack = IconPack.load(context, packageName) ?: return emptyList()
    return mainApps.asSequence()
        .mapNotNull { pack.drawableFor(it.key.packageName, it.key.className, density) }
        .take(PREVIEW_COUNT)
        .map { it.toBitmap(PREVIEW_PX, PREVIEW_PX).asImageBitmap() }
        .toList()
}
