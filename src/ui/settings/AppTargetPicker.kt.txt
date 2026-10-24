package io.github.jacomartins.canyonui.ui.settings

import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.os.Build
import androidx.compose.foundation.Image
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.heightIn
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.RadioButton
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.produceState
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import io.github.jacomartins.canyonui.ui.design.SectionHeader
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.unit.dp
import androidx.core.graphics.drawable.toBitmap
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.data.AppInfo
import io.github.jacomartins.canyonui.ui.design.SecondaryButton
import io.github.jacomartins.canyonui.ui.design.PrimaryButton
import androidx.compose.foundation.text.KeyboardActions
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.foundation.text.BasicTextField
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.height
import androidx.compose.ui.graphics.SolidColor
import io.github.jacomartins.canyonui.ui.design.oneDepthBorder
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

/**
 * Something a gesture or button can open: [key] is "action|package/class". [special] marks the
 * apps found by the picker's own actions (search apps, wallpaper apps).
 */
class AppTarget(val key: String, val label: String, val icon: ImageBitmap?, val special: Boolean)

/** Actions search apps answer to; a phone's own search (no app icon) is found this way. */
val SEARCH_ACTIONS = listOf("android.search.action.GLOBAL_SEARCH", Intent.ACTION_WEB_SEARCH)

/** Apps that set wallpapers, including ones without an app icon. */
val WALLPAPER_ACTIONS = listOf(Intent.ACTION_SET_WALLPAPER, android.app.WallpaperManager.ACTION_LIVE_WALLPAPER_CHOOSER)

/** A target typed as a package name ("pkg:" + package), for apps without an icon to pick. */
const val PACKAGE_TARGET = "pkg:"

/** The package a stored target opens. */
fun targetPackage(target: String): String =
    if (target.startsWith(PACKAGE_TARGET)) target.removePrefix(PACKAGE_TARGET) else target.substringAfter('|').substringBefore('/')

/** The app's name for [target], or its package when it is not installed. */
fun targetLabel(context: Context, target: String): String {
    val pkg = targetPackage(target)
    return runCatching { context.packageManager.getApplicationLabel(context.packageManager.getApplicationInfo(pkg, 0)).toString() }.getOrDefault(pkg)
}

/**
 * The intent that opens [target]. A typed package is tried with [actions] first (a search or
 * wallpaper screen it offers), then its launcher entry, then any screen it lets others open.
 */
fun targetIntent(context: Context, target: String, actions: List<String>): Intent? {
    if (!target.startsWith(PACKAGE_TARGET)) {
        val action = target.substringBefore('|')
        val cn = android.content.ComponentName.unflattenFromString(target.substringAfter('|')) ?: return null
        return Intent(action).setComponent(cn).apply { if (action == Intent.ACTION_MAIN) addCategory(Intent.CATEGORY_LAUNCHER) }
    }
    val pkg = target.removePrefix(PACKAGE_TARGET).trim()
    if (pkg.isEmpty()) return null
    val pm = context.packageManager
    for (action in actions) {
        val i = Intent(action).setPackage(pkg)
        if (i.resolveActivity(pm) != null) return i
    }
    pm.getLaunchIntentForPackage(pkg)?.let { return it }
    val activities = try {
        @Suppress("DEPRECATION")
        pm.getPackageInfo(pkg, PackageManager.GET_ACTIVITIES).activities
    } catch (e: PackageManager.NameNotFoundException) {
        null
    }
    val open = activities?.firstOrNull { it.exported && it.enabled && it.permission == null } ?: return null
    return Intent(Intent.ACTION_MAIN).setClassName(pkg, open.name)
}

/** Apps answering [actions] first, then every app (main profile), without duplicates. */
fun appTargets(context: Context, apps: List<AppInfo>, actions: List<String>): List<AppTarget> {
    val pm = context.packageManager
    val px = (40 * context.resources.displayMetrics.density).toInt()
    val seen = HashSet<String>()
    val out = ArrayList<AppTarget>()
    for (action in actions) {
        val found = if (Build.VERSION.SDK_INT >= 33) {
            pm.queryIntentActivities(Intent(action), PackageManager.ResolveInfoFlags.of(0))
        } else {
            @Suppress("DEPRECATION")
            pm.queryIntentActivities(Intent(action), 0)
        }
        for (ri in found) {
            val cn = android.content.ComponentName(ri.activityInfo.packageName, ri.activityInfo.name)
            if (!seen.add(cn.packageName)) continue
            val icon = try {
                ri.loadIcon(pm).toBitmap(px, px).asImageBitmap()
            } catch (e: RuntimeException) {
                null
            }
            out += AppTarget("$action|${cn.flattenToString()}", ri.loadLabel(pm).toString(), icon, special = true)
        }
    }
    for (a in apps) {
        if (!a.isMainProfile || a.key.packageName in seen) continue
        out += AppTarget("${Intent.ACTION_MAIN}|${a.component.flattenToString()}", a.label, a.image, special = false)
    }
    return out
}

/**
 * Chooses an app for a gesture or button. [noneLabel] is the first choice (selected = null);
 * [specialHeader] titles the apps found through [actions].
 */
@Composable
fun AppTargetPicker(
    title: String,
    noneLabel: String,
    specialHeader: String,
    actions: List<String>,
    selected: String?,
    apps: List<AppInfo>,
    onSelect: (String?) -> Unit,
    onDismiss: () -> Unit,
) {
    val context = LocalContext.current
    val targets by produceState<List<AppTarget>?>(null, apps, actions) {
        value = withContext(Dispatchers.Default) { appTargets(context, apps, actions) }
    }
    var typed by remember { mutableStateOf(selected?.takeIf { it.startsWith(PACKAGE_TARGET) }?.removePrefix(PACKAGE_TARGET).orEmpty()) }
    io.github.jacomartins.canyonui.ui.design.SheetDialog(
        onDismissRequest = onDismiss,
        confirmButton = { SecondaryButton(stringResource(R.string.cancel), onDismiss) },
        title = { Text(title) },
        text = {
            val list = targets.orEmpty()
            LazyColumn(Modifier.heightIn(max = 480.dp)) {
                item { Choice(noneLabel, null, selected == null) { onSelect(null) } }
                // Any package, by name: for apps that have no icon to pick from the lists.
                item {
                    Header(stringResource(R.string.target_package))
                    Row(verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        // A pill field with the interface's border, like the search field.
                        val colors = MaterialTheme.colorScheme
                        androidx.compose.material3.Surface(
                            shape = CircleShape,
                            color = colors.surfaceContainerHighest,
                            modifier = Modifier.weight(1f).height(44.dp).oneDepthBorder(),
                        ) {
                            Box(Modifier.padding(horizontal = 16.dp), contentAlignment = Alignment.CenterStart) {
                                if (typed.isEmpty()) Text(stringResource(R.string.target_package_hint), color = colors.onSurfaceVariant, style = MaterialTheme.typography.bodyLarge)
                                BasicTextField(
                                    value = typed,
                                    onValueChange = { typed = it.trim() },
                                    singleLine = true,
                                    textStyle = MaterialTheme.typography.bodyLarge.copy(color = colors.onSurface),
                                    cursorBrush = SolidColor(colors.primary),
                                    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Uri, imeAction = ImeAction.Done),
                                    keyboardActions = KeyboardActions(onDone = { if (typed.isNotEmpty()) onSelect(PACKAGE_TARGET + typed) }),
                                    modifier = Modifier.fillMaxWidth(),
                                )
                            }
                        }
                        PrimaryButton(stringResource(R.string.target_package_use), { onSelect(PACKAGE_TARGET + typed) }, enabled = typed.isNotEmpty())
                    }
                }
                val special = list.filter { it.special }
                if (special.isNotEmpty()) {
                    item { Header(specialHeader) }
                    items(special, key = { it.key }) { t -> Choice(t.label, t.icon, selected == t.key) { onSelect(t.key) } }
                }
                item { Header(stringResource(R.string.swipe_up_all_apps)) }
                items(list.filter { !it.special }, key = { it.key }) { t -> Choice(t.label, t.icon, selected == t.key) { onSelect(t.key) } }
            }
        },
    )
}

@Composable
private fun Header(text: String) {
    SectionHeader(text, Modifier.padding(top = 12.dp, bottom = 8.dp, start = 4.dp))
}

@Composable
private fun Choice(label: String, icon: ImageBitmap?, selected: Boolean, onClick: () -> Unit) {
    Row(
        Modifier
            .fillMaxWidth()
            .clip(CircleShape)
            .clickable(role = Role.RadioButton, onClick = onClick)
            .padding(horizontal = 8.dp, vertical = 6.dp),
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.spacedBy(12.dp),
    ) {
        RadioButton(selected = selected, onClick = null)
        if (icon != null) Image(icon, contentDescription = null, modifier = Modifier.size(32.dp)) else Spacer(Modifier.size(32.dp))
        Text(label, style = MaterialTheme.typography.bodyLarge, maxLines = 1)
    }
}
