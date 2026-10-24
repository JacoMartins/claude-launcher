package io.github.jacomartins.canyonui

import android.Manifest
import android.graphics.RectF
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.background
import androidx.compose.foundation.gestures.detectTransformGestures
import androidx.compose.foundation.layout.navigationBars
import androidx.compose.foundation.layout.statusBars
import androidx.compose.runtime.mutableFloatStateOf
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.FilterQuality
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.layout.onSizeChanged
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.IntOffset
import androidx.compose.ui.unit.IntSize
import kotlin.math.max
import kotlin.math.roundToInt
import android.app.WallpaperInfo
import android.app.WallpaperManager
import android.content.ActivityNotFoundException
import android.content.ComponentName
import android.content.ContentUris
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.provider.MediaStore
import android.service.wallpaper.WallpaperService
import android.util.Size
import android.widget.Toast
import androidx.activity.ComponentActivity
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.activity.result.PickVisualMediaRequest
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.Image
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.aspectRatio
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.safeDrawing
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.windowInsetsPadding
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.produceState
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.ImageBitmap
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.unit.dp
import androidx.core.content.ContextCompat
import androidx.core.graphics.drawable.toBitmap
import io.github.jacomartins.canyonui.ui.design.PrimaryButton
import io.github.jacomartins.canyonui.ui.design.SecondaryButton
import io.github.jacomartins.canyonui.ui.design.SettingsSection
import io.github.jacomartins.canyonui.ui.design.Spacing
import io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop
import io.github.jacomartins.canyonui.ui.design.settingsPageColor
import io.github.jacomartins.canyonui.ui.theme.HomeTheme
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

/**
 * Canyon UI's own wallpaper chooser (the edit mode's Wallpaper button):
 * - Photos: the photo picker or another gallery app, the latest photos and their folders (when
 *   photo access is allowed). A photo then gets the crop screen, to choose the part that shows.
 * - Live wallpapers: installed ones; Android's own preview sets them.
 * - Wallpaper apps: the phone maker's and others, which set the wallpaper themselves.
 *
 * A photo set on the home screen from here is also what the frosted and glass effects blur.
 * Once a wallpaper is set (here, or by a wallpaper app opened from here), it goes back home.
 */
class WallpaperActivity : ComponentActivity() {

    // A wallpaper set by an app opened from here; one set here goes home once it is done.
    private val colorsListener = WallpaperManager.OnColorsChangedListener { _, _ -> if (!WallpaperBackdrop.setting) goHome() }

    /** Back to the home screen, to see the new wallpaper. */
    fun goHome() {
        if (isFinishing) return
        startActivity(Intent(this, HomeActivity::class.java).addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_NEW_TASK))
        finish()
    }


    override fun onDestroy() {
        WallpaperManager.getInstance(this).removeOnColorsChangedListener(colorsListener)
        super.onDestroy()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        enableEdgeToEdge()
        super.onCreate(savedInstanceState)
        // Kept while this screen exists, also while a wallpaper app opened from here is in front.
        WallpaperManager.getInstance(this).addOnColorsChangedListener(colorsListener, android.os.Handler(mainLooper))
        val app = application as LauncherApp
        setContent {
            val settings by app.settings.settings.collectAsState(initial = null)
            HomeTheme(phonePalette = settings?.phonePalette == true) {
                Surface(Modifier.fillMaxSize(), color = settingsPageColor()) {
                    WallpaperScreen(onDone = ::goHome)
                }
            }
        }
    }
}

/** A folder of photos (MediaStore bucket): its name, newest photo and how many it holds. */
private class PhotoFolder(val id: String, val name: String, val cover: ImageBitmap?, val count: Int)

/** An app the "Add from" list offers: [pick] apps hand back an image, the others set it themselves. */
private class SourceApp(val intent: Intent, val label: String, val icon: ImageBitmap?, val pick: Boolean)

/** A live wallpaper: its service and what to show for it. */
private class LiveWallpaper(val component: ComponentName, val label: String, val thumb: ImageBitmap?)

private val READ_IMAGES = if (Build.VERSION.SDK_INT >= 33) Manifest.permission.READ_MEDIA_IMAGES else Manifest.permission.READ_EXTERNAL_STORAGE

private fun canReadPhotos(context: Context): Boolean {
    if (ContextCompat.checkSelfPermission(context, READ_IMAGES) == PackageManager.PERMISSION_GRANTED) return true
    // Android 14+: access to the photos the user picked in the permission prompt.
    return Build.VERSION.SDK_INT >= 34 &&
        ContextCompat.checkSelfPermission(context, Manifest.permission.READ_MEDIA_VISUAL_USER_SELECTED) == PackageManager.PERMISSION_GRANTED
}

@Composable
private fun WallpaperScreen(onDone: () -> Unit) {
    val context = LocalContext.current
    // A photo chosen: first the part of it to show, then where it goes.
    var chosen by remember { mutableStateOf<Uri?>(null) }
    var cropped by remember { mutableStateOf<RectF?>(null) }
    var canRead by remember { mutableStateOf(canReadPhotos(context)) }
    var asked by remember { mutableStateOf(false) }
    val picker = rememberLauncherForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri -> if (uri != null) chosen = uri }
    // An image handed back by a gallery or file app from "Other apps".
    val fromApp = rememberLauncherForActivityResult(ActivityResultContracts.StartActivityForResult()) { r -> r.data?.data?.let { chosen = it } }
    var showSources by remember { mutableStateOf(false) }
    var folder by remember { mutableStateOf<PhotoFolder?>(null) }
    val permission = rememberLauncherForActivityResult(ActivityResultContracts.RequestMultiplePermissions()) {
        asked = true
        canRead = canReadPhotos(context)
    }
    val recent by produceState(emptyList<Pair<Uri, ImageBitmap>>(), canRead) {
        value = if (canRead) withContext(Dispatchers.IO) { recentPhotos(context) } else emptyList()
    }
    val folders by produceState(emptyList<PhotoFolder>(), canRead) {
        value = if (canRead) withContext(Dispatchers.IO) { photoFolders(context) } else emptyList()
    }
    val live by produceState(emptyList<LiveWallpaper>()) {
        value = withContext(Dispatchers.IO) { liveWallpapers(context) }
    }
    val apps by produceState(emptyList<SourceApp>()) { value = withContext(Dispatchers.IO) { sourceApps(context) } }

    val photo = chosen
    if (photo != null) {
        CropScreen(photo, onCancel = { chosen = null }, onConfirm = { cropped = it })
        cropped?.let { crop ->
            SetDialog(photo, crop, onDone, onDismiss = { cropped = null }, clear = { cropped = null; chosen = null })
        }
        return
    }

    // One folder open: its photos, newest first; back returns to the chooser.
    folder?.let { f ->
        androidx.activity.compose.BackHandler { folder = null }
        val photos by produceState(emptyList<Pair<Uri, ImageBitmap>>(), f.id) {
            value = withContext(Dispatchers.IO) { recentPhotos(context, f.id, FOLDER_LIMIT) }
        }
        Column(
            Modifier.fillMaxSize().windowInsetsPadding(WindowInsets.safeDrawing).verticalScroll(rememberScrollState()),
            verticalArrangement = Arrangement.spacedBy(Spacing.sm),
        ) {
            Text(f.name, style = MaterialTheme.typography.headlineMedium, modifier = Modifier.padding(start = Spacing.mdLg, top = Spacing.lg, bottom = Spacing.md))
            SettingsSection {
                ThumbGrid(photos.map { it.second }, aspect = 1f) { i -> chosen = photos[i].first }
            }
        }
        return
    }

    Column(
        Modifier
            .fillMaxSize()
            .windowInsetsPadding(WindowInsets.safeDrawing)
            .verticalScroll(rememberScrollState()),
        verticalArrangement = Arrangement.spacedBy(Spacing.sm),
    ) {
        Text(
            stringResource(R.string.choose_wallpaper),
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.padding(start = Spacing.mdLg, top = Spacing.lg, bottom = Spacing.md),
        )

        // Photos: where to pick from, then the latest ones, then their folders.
        SettingsSection(title = stringResource(R.string.wallpaper_photos)) {
            Row(horizontalArrangement = Arrangement.spacedBy(Spacing.sm)) {
                PrimaryButton(
                    stringResource(R.string.wallpaper_photos),
                    { picker.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)) },
                    Modifier.weight(1f),
                    icon = R.drawable.ph_wallpaper,
                )
                SecondaryButton(stringResource(R.string.wallpaper_other_apps), { showSources = true }, Modifier.weight(1f))
            }
            if (recent.isNotEmpty()) {
                io.github.jacomartins.canyonui.ui.design.SectionHeader(stringResource(R.string.wallpaper_recent), Modifier.padding(top = Spacing.sm))
                ThumbGrid(recent.map { it.second }, aspect = 1f) { i -> chosen = recent[i].first }
            }
            if (folders.isNotEmpty()) {
                io.github.jacomartins.canyonui.ui.design.SectionHeader(stringResource(R.string.wallpaper_folders), Modifier.padding(top = Spacing.sm))
                FolderGrid(folders) { folder = it }
            }
            if (recent.isEmpty() && !canRead) {
                if (asked) {
                    Text(stringResource(R.string.wallpaper_recent_denied), style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.onSurfaceVariant)
                } else {
                    SecondaryButton(stringResource(R.string.wallpaper_show_recent), {
                        val perms = buildList {
                            add(READ_IMAGES)
                            if (Build.VERSION.SDK_INT >= 34) add(Manifest.permission.READ_MEDIA_VISUAL_USER_SELECTED)
                        }
                        permission.launch(perms.toTypedArray())
                    }, Modifier.fillMaxWidth())
                }
            }
        }

        // Live wallpapers: tiles with their previews; Android's own preview sets them.
        if (live.isNotEmpty()) {
            SettingsSection(title = stringResource(R.string.wallpaper_live)) {
                LiveGrid(live) { openLive(context, it.component) }
            }
        }

        // Wallpaper apps (the phone maker's, others): they set the wallpaper themselves.
        val setters = apps.filter { !it.pick }
        if (setters.isNotEmpty()) {
            SettingsSection(title = stringResource(R.string.wallpaper_apps)) {
                Column(verticalArrangement = Arrangement.spacedBy(Spacing.xs)) {
                    for (a in setters) SourceRow(a.label, a.icon) { openApp(context, a, fromApp::launch) }
                }
            }
        }
    }

    if (showSources) {
        SourcesDialog(
            apps.filter { it.pick },
            onApp = { app ->
                showSources = false
                openApp(context, app, fromApp::launch)
            },
            onDismiss = { showSources = false },
        )
    }
}

/** Opens [app]: one that hands back an image through [pick], or a wallpaper app. */
private fun openApp(context: Context, app: SourceApp, pick: (Intent) -> Unit) {
    try {
        if (app.pick) pick(app.intent) else context.startActivity(app.intent)
    } catch (e: android.content.ActivityNotFoundException) {
        Toast.makeText(context, R.string.wallpaper_live_unavailable, Toast.LENGTH_SHORT).show()
    } catch (e: SecurityException) {
        Toast.makeText(context, R.string.wallpaper_live_unavailable, Toast.LENGTH_SHORT).show()
    }
}

/**
 * The chosen photo over the whole screen, as it will look: drag and pinch to pick the part that
 * shows. It always covers the screen (no empty edges).
 */
@Composable
private fun CropScreen(uri: Uri, onCancel: () -> Unit, onConfirm: (RectF) -> Unit) {
    androidx.activity.compose.BackHandler(onBack = onCancel)
    val context = LocalContext.current
    val image by produceState<ImageBitmap?>(null, uri) { value = withContext(Dispatchers.IO) { previewImage(context, uri) } }
    var box by remember { mutableStateOf(IntSize.Zero) }
    var zoom by remember(uri) { mutableFloatStateOf(1f) }
    var offset by remember(uri) { mutableStateOf(Offset.Unspecified) }
    Box(Modifier.fillMaxSize().background(Color.Black).onSizeChanged { box = it }) {
        val img = image
        if (img != null && box.width > 0 && box.height > 0) {
            // The smallest scale that still covers the screen; zooming goes up from there.
            val base = max(box.width / img.width.toFloat(), box.height / img.height.toFloat())
            fun clamp(o: Offset, s: Float) = Offset(
                o.x.coerceIn(box.width - img.width * s, 0f),
                o.y.coerceIn(box.height - img.height * s, 0f),
            )
            if (offset == Offset.Unspecified) offset = Offset((box.width - img.width * base) / 2f, (box.height - img.height * base) / 2f)
            Canvas(
                Modifier
                    .fillMaxSize()
                    .pointerInput(img, box) {
                        detectTransformGestures { centroid, pan, gestureZoom, _ ->
                            val s = base * zoom
                            val nz = (zoom * gestureZoom).coerceIn(1f, MAX_ZOOM)
                            val ns = base * nz
                            // The point under the fingers stays under them while zooming.
                            val p = (centroid - offset) / s
                            offset = clamp(centroid - p * ns + pan, ns)
                            zoom = nz
                        }
                    },
            ) {
                val s = base * zoom
                drawImage(
                    img,
                    dstOffset = IntOffset(offset.x.roundToInt(), offset.y.roundToInt()),
                    dstSize = IntSize((img.width * s).roundToInt(), (img.height * s).roundToInt()),
                    filterQuality = FilterQuality.High,
                )
            }
            Text(
                stringResource(R.string.wallpaper_crop_hint),
                style = MaterialTheme.typography.bodyMedium,
                color = Color.White,
                textAlign = TextAlign.Center,
                modifier = Modifier
                    .align(Alignment.TopCenter)
                    .windowInsetsPadding(WindowInsets.statusBars)
                    .padding(top = Spacing.md, start = Spacing.lg, end = Spacing.lg)
                    .clip(RoundedCornerShape(50))
                    .background(Color.Black.copy(alpha = 0.45f))
                    .padding(horizontal = Spacing.md, vertical = Spacing.xs),
            )
            Row(
                Modifier
                    .align(Alignment.BottomCenter)
                    .windowInsetsPadding(WindowInsets.navigationBars)
                    .padding(Spacing.lg),
                horizontalArrangement = Arrangement.spacedBy(Spacing.sm),
            ) {
                SecondaryButton(stringResource(R.string.cancel), onCancel, Modifier.weight(1f))
                PrimaryButton(stringResource(R.string.wallpaper_crop_set), {
                    val s = base * zoom
                    val l = -offset.x / (img.width * s)
                    val t = -offset.y / (img.height * s)
                    onConfirm(RectF(l, t, l + box.width / (img.width * s), t + box.height / (img.height * s)))
                }, Modifier.weight(1f))
            }
        }
    }
}

/** [uri] decoded at about twice the screen's size: sharp when zoomed, far less than a full photo. */
private fun previewImage(context: Context, uri: Uri): ImageBitmap? = try {
    val (w, h) = WallpaperBackdrop.screenSize(context)
    android.graphics.ImageDecoder.decodeBitmap(android.graphics.ImageDecoder.createSource(context.contentResolver, uri)) { decoder, info, _ ->
        val s = max(2f * w / info.size.width, 2f * h / info.size.height).coerceAtMost(1f)
        decoder.setTargetSize((info.size.width * s).roundToInt().coerceAtLeast(1), (info.size.height * s).roundToInt().coerceAtLeast(1))
    }.asImageBitmap()
} catch (e: Exception) {
    null
}

/** Asks where the photo goes, sets the [crop] part of it, then goes home. */
@Composable
private fun SetDialog(uri: Uri, crop: RectF, onDone: () -> Unit, onDismiss: () -> Unit, clear: () -> Unit) {
    val context = LocalContext.current
    WhereDialog(
        onPick = { which ->
            clear()
            // In the app's scope, not this dialog's: closing the dialog cancelled the set half
            // way (the wallpaper changed, but the blur was never rebuilt and home never came).
            (context.applicationContext as LauncherApp).scope.launch {
                val ok = WallpaperBackdrop.setFromPhoto(context.applicationContext, uri, which, crop)
                Toast.makeText(context.applicationContext, if (ok) R.string.wallpaper_done else R.string.render_photo_failed, Toast.LENGTH_SHORT).show()
                if (ok) onDone()
            }
        },
        onDismiss = onDismiss,
    )
}

/** "Other apps": galleries and file managers that hand back an image. */
@Composable
private fun SourcesDialog(apps: List<SourceApp>, onApp: (SourceApp) -> Unit, onDismiss: () -> Unit) {
    io.github.jacomartins.canyonui.ui.design.SheetDialog(
        onDismissRequest = onDismiss,
        title = { Text(stringResource(R.string.wallpaper_other_apps)) },
        text = {
            Column(Modifier.verticalScroll(rememberScrollState()), verticalArrangement = Arrangement.spacedBy(Spacing.xs)) {
                for (a in apps) SourceRow(a.label, a.icon) { onApp(a) }
            }
        },
        confirmButton = {},
        dismissButton = { SecondaryButton(stringResource(R.string.cancel), onDismiss) },
    )
}

@Composable
private fun SourceRow(label: String, icon: ImageBitmap?, onClick: () -> Unit) {
    Row(
        Modifier.fillMaxWidth().clip(RoundedCornerShape(16.dp)).clickable(role = Role.Button, onClick = onClick).padding(vertical = Spacing.sm, horizontal = Spacing.xs),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Box(Modifier.size(36.dp), contentAlignment = Alignment.Center) {
            if (icon != null) {
                Image(icon, contentDescription = null, modifier = Modifier.fillMaxSize())
            } else {
                androidx.compose.material3.Icon(androidx.compose.ui.res.painterResource(R.drawable.ph_wallpaper), contentDescription = null, tint = MaterialTheme.colorScheme.onSurface)
            }
        }
        Text(label, style = MaterialTheme.typography.titleMedium, modifier = Modifier.padding(start = Spacing.smMd))
    }
}

/** Photo folders, two a row, with their newest photo and their name. */
@Composable
private fun FolderGrid(folders: List<PhotoFolder>, onOpen: (PhotoFolder) -> Unit) {
    Column(verticalArrangement = Arrangement.spacedBy(Spacing.sm)) {
        for (row in folders.chunked(2)) {
            Row(horizontalArrangement = Arrangement.spacedBy(Spacing.sm)) {
                for (f in row) {
                    Column(
                        Modifier.weight(1f).clip(RoundedCornerShape(16.dp)).clickable(role = Role.Button) { onOpen(f) },
                    ) {
                        Box(Modifier.fillMaxWidth().aspectRatio(1f).clip(RoundedCornerShape(16.dp))) {
                            f.cover?.let { Image(it, contentDescription = null, contentScale = ContentScale.Crop, modifier = Modifier.fillMaxSize()) }
                        }
                        Text(f.name, style = MaterialTheme.typography.titleSmall, maxLines = 1, modifier = Modifier.padding(top = Spacing.xs, start = Spacing.xs))
                        Text("${f.count}", style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.onSurfaceVariant, modifier = Modifier.padding(start = Spacing.xs))
                    }
                }
                if (row.size == 1) Box(Modifier.weight(1f))
            }
        }
    }
}

/** Live wallpapers, two a row: their preview (screen-shaped) and name. */
@Composable
private fun LiveGrid(items: List<LiveWallpaper>, onOpen: (LiveWallpaper) -> Unit) {
    Column(verticalArrangement = Arrangement.spacedBy(Spacing.sm)) {
        for (row in items.chunked(2)) {
            Row(horizontalArrangement = Arrangement.spacedBy(Spacing.sm)) {
                for (w in row) {
                    Column(Modifier.weight(1f).clip(RoundedCornerShape(16.dp)).clickable(role = Role.Button) { onOpen(w) }) {
                        Box(
                            Modifier.fillMaxWidth().aspectRatio(0.75f).clip(RoundedCornerShape(16.dp))
                                .background(MaterialTheme.colorScheme.surfaceContainerHighest),
                        ) {
                            w.thumb?.let { Image(it, contentDescription = null, contentScale = ContentScale.Crop, modifier = Modifier.fillMaxSize()) }
                        }
                        Text(w.label, style = MaterialTheme.typography.titleSmall, maxLines = 1, modifier = Modifier.padding(top = Spacing.xs, start = Spacing.xs))
                    }
                }
                if (row.size == 1) Box(Modifier.weight(1f))
            }
        }
    }
}

/** Three photos a row, rounded, tappable. */
@Composable
private fun ThumbGrid(images: List<ImageBitmap>, aspect: Float, onPick: (Int) -> Unit) {
    Column(verticalArrangement = Arrangement.spacedBy(Spacing.sm)) {
        for (row in images.indices.chunked(3)) {
            Row(horizontalArrangement = Arrangement.spacedBy(Spacing.sm)) {
                for (i in row) {
                    Image(
                        images[i],
                        contentDescription = null,
                        contentScale = ContentScale.Crop,
                        modifier = Modifier
                            .weight(1f)
                            .aspectRatio(aspect)
                            .clip(RoundedCornerShape(16.dp))
                            .clickable(role = Role.Button) { onPick(i) },
                    )
                }
                repeat(3 - row.size) { Box(Modifier.weight(1f)) }
            }
        }
    }
}

/** Home, lock or both. */
@Composable
private fun WhereDialog(onPick: (Int) -> Unit, onDismiss: () -> Unit) {
    io.github.jacomartins.canyonui.ui.design.SheetDialog(
        onDismissRequest = onDismiss,
        title = { Text(stringResource(R.string.wallpaper_set_where)) },
        text = {
            Column(verticalArrangement = Arrangement.spacedBy(Spacing.sm)) {
                val home = stringResource(R.string.wallpaper_home)
                PrimaryButton(home, { onPick(WallpaperManager.FLAG_SYSTEM) }, Modifier.fillMaxWidth().semantics { contentDescription = home })
                SecondaryButton(stringResource(R.string.wallpaper_lock), { onPick(WallpaperManager.FLAG_LOCK) }, Modifier.fillMaxWidth())
                SecondaryButton(stringResource(R.string.wallpaper_both), { onPick(WallpaperManager.FLAG_SYSTEM or WallpaperManager.FLAG_LOCK) }, Modifier.fillMaxWidth())
            }
        },
        confirmButton = {},
        dismissButton = { SecondaryButton(stringResource(R.string.cancel), onDismiss) },
    )
}

/** The latest photos (needs photo access), of one folder ([bucket]) or all, as small thumbnails. */
private fun recentPhotos(context: Context, bucket: String? = null, limit: Int = RECENT_COUNT): List<Pair<Uri, ImageBitmap>> {
    val out = ArrayList<Pair<Uri, ImageBitmap>>()
    val resolver = context.contentResolver
    val collection = MediaStore.Images.Media.EXTERNAL_CONTENT_URI
    val args = Bundle().apply {
        putStringArray(android.content.ContentResolver.QUERY_ARG_SORT_COLUMNS, arrayOf(MediaStore.Images.Media.DATE_ADDED))
        putInt(android.content.ContentResolver.QUERY_ARG_SORT_DIRECTION, android.content.ContentResolver.QUERY_SORT_DIRECTION_DESCENDING)
        putInt(android.content.ContentResolver.QUERY_ARG_LIMIT, limit)
        if (bucket != null) {
            putString(android.content.ContentResolver.QUERY_ARG_SQL_SELECTION, "${MediaStore.Images.Media.BUCKET_ID} = ?")
            putStringArray(android.content.ContentResolver.QUERY_ARG_SQL_SELECTION_ARGS, arrayOf(bucket))
        }
    }
    try {
        resolver.query(collection, arrayOf(MediaStore.Images.Media._ID), args, null)?.use { c ->
            val idCol = c.getColumnIndexOrThrow(MediaStore.Images.Media._ID)
            while (c.moveToNext()) {
                val uri = ContentUris.withAppendedId(collection, c.getLong(idCol))
                val thumb = try {
                    resolver.loadThumbnail(uri, Size(THUMB_PX, THUMB_PX), null).asImageBitmap()
                } catch (e: Exception) {
                    continue
                }
                out += uri to thumb
            }
        }
    } catch (e: SecurityException) {
        return emptyList()
    }
    return out
}

/** The folders holding photos, newest first, each with its newest photo as cover. */
private fun photoFolders(context: Context): List<PhotoFolder> {
    val resolver = context.contentResolver
    val collection = MediaStore.Images.Media.EXTERNAL_CONTENT_URI
    val firsts = LinkedHashMap<String, Pair<String, Long>>()
    val counts = HashMap<String, Int>()
    val args = Bundle().apply {
        putStringArray(android.content.ContentResolver.QUERY_ARG_SORT_COLUMNS, arrayOf(MediaStore.Images.Media.DATE_ADDED))
        putInt(android.content.ContentResolver.QUERY_ARG_SORT_DIRECTION, android.content.ContentResolver.QUERY_SORT_DIRECTION_DESCENDING)
    }
    val cols = arrayOf(MediaStore.Images.Media._ID, MediaStore.Images.Media.BUCKET_ID, MediaStore.Images.Media.BUCKET_DISPLAY_NAME)
    try {
        resolver.query(collection, cols, args, null)?.use { c ->
            val idCol = c.getColumnIndexOrThrow(MediaStore.Images.Media._ID)
            val bCol = c.getColumnIndexOrThrow(MediaStore.Images.Media.BUCKET_ID)
            val nCol = c.getColumnIndexOrThrow(MediaStore.Images.Media.BUCKET_DISPLAY_NAME)
            while (c.moveToNext()) {
                val b = c.getString(bCol) ?: continue
                counts[b] = (counts[b] ?: 0) + 1
                if (b !in firsts) firsts[b] = (c.getString(nCol) ?: "") to c.getLong(idCol)
            }
        }
    } catch (e: SecurityException) {
        return emptyList()
    }
    return firsts.entries.take(FOLDER_COUNT).map { (b, v) ->
        val cover = try {
            resolver.loadThumbnail(ContentUris.withAppendedId(collection, v.second), Size(THUMB_PX, THUMB_PX), null).asImageBitmap()
        } catch (e: Exception) {
            null
        }
        PhotoFolder(b, v.first, cover, counts[b] ?: 0)
    }
}

/** Apps that hand back an image (galleries, file managers), then wallpaper apps; not this one. */
private fun sourceApps(context: Context): List<SourceApp> {
    val pm = context.packageManager
    val px = (36 * context.resources.displayMetrics.density).toInt()
    val seen = HashSet<String>()
    val out = ArrayList<SourceApp>()
    fun add(base: Intent, pick: Boolean) {
        @Suppress("DEPRECATION")
        for (ri in pm.queryIntentActivities(base, 0)) {
            val a = ri.activityInfo
            if (a.packageName == context.packageName || !seen.add(a.packageName + pick)) continue
            val icon = try { ri.loadIcon(pm).toBitmap(px, px).asImageBitmap() } catch (e: RuntimeException) { null }
            out += SourceApp(Intent(base).setClassName(a.packageName, a.name), ri.loadLabel(pm).toString(), icon, pick)
        }
    }
    add(Intent(Intent.ACTION_GET_CONTENT).setType("image/*").addCategory(Intent.CATEGORY_OPENABLE), pick = true)
    add(Intent(Intent.ACTION_SET_WALLPAPER), pick = false)
    add(Intent(WallpaperManager.ACTION_LIVE_WALLPAPER_CHOOSER), pick = false)
    return out.sortedWith(compareBy({ !it.pick }, { it.label.lowercase() }))
}

/** Installed live wallpapers, by name. */
private fun liveWallpapers(context: Context): List<LiveWallpaper> {
    val pm = context.packageManager
    val services = pm.queryIntentServices(Intent(WallpaperService.SERVICE_INTERFACE), PackageManager.GET_META_DATA)
    return services.mapNotNull { ri ->
        try {
            val info = WallpaperInfo(context, ri)
            val thumb = info.loadThumbnail(pm)?.let { d ->
                d.toBitmap(THUMB_PX, (THUMB_PX * d.intrinsicHeight.coerceAtLeast(1) / d.intrinsicWidth.coerceAtLeast(1)).coerceAtLeast(1)).asImageBitmap()
            }
            LiveWallpaper(info.component, info.loadLabel(pm).toString(), thumb)
        } catch (e: Exception) {
            null
        }
    }.sortedBy { it.label.lowercase() }
}

/** Android's preview of [component], where it is set. */
private fun openLive(context: Context, component: ComponentName) {
    val intent = Intent(WallpaperManager.ACTION_CHANGE_LIVE_WALLPAPER)
        .putExtra(WallpaperManager.EXTRA_LIVE_WALLPAPER_COMPONENT, component)
    try {
        context.startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        Toast.makeText(context, R.string.wallpaper_live_unavailable, Toast.LENGTH_SHORT).show()
    } catch (e: SecurityException) {
        Toast.makeText(context, R.string.wallpaper_live_unavailable, Toast.LENGTH_SHORT).show()
    }
}

private const val RECENT_COUNT = 12
private const val FOLDER_COUNT = 20
private const val FOLDER_LIMIT = 120
private const val THUMB_PX = 256

/** How far the crop screen zooms in. */
private const val MAX_ZOOM = 5f
