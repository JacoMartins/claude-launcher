package io.github.jacomartins.canyonui.ui.overlay

import android.graphics.Rect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableFloatStateOf
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.mutableLongStateOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.compose.ui.graphics.ImageBitmap
import io.github.jacomartins.canyonui.ui.design.MenuEntry
import io.github.jacomartins.canyonui.ui.widget.PickerWidget
import io.github.jacomartins.canyonui.data.AppEntry
import io.github.jacomartins.canyonui.data.AppInfo
import io.github.jacomartins.canyonui.data.FolderEntry
import io.github.jacomartins.canyonui.data.IconOverride


/** A long-press menu next to [anchor] (window coordinates). */
class ItemMenu(val anchor: Rect, val entries: List<MenuEntry>)

class Snack(val message: String, val onUndo: (() -> Unit)?) {
    val id = System.nanoTime()
}

/** What the overlay asks the activity to do. */
interface OverlayActions {
    fun launchFromFolder(app: AppEntry, bounds: Rect?)
    fun armDragFromFolder(app: AppEntry, folder: FolderEntry, bounds: Rect)
    fun renameFolder(folder: FolderEntry, title: String)
    fun setFolderColor(folder: FolderEntry, color: Int)
    fun closeFolder()
    fun chooseWallpaper()
    fun openWidgets()
    fun closeEditApp()
    fun saveEditApp(app: AppInfo, override: IconOverride)
    fun pickIconImage()
    fun closePicker()
    fun addWidget(widget: PickerWidget)
    fun armWidgetDrag(widget: PickerWidget, preview: ImageBitmap?, bounds: Rect)
    fun openSettings()
    fun addPage()
    fun deletePage(index: Int)
    fun makeHomePage(index: Int)
    fun categorizeApps()
    fun movePage(from: Int, to: Int)
}

/**
 * Everything the overlay shows, as Compose state written from the activity on the main thread.
 */
class OverlayState {
    var menu by mutableStateOf<ItemMenu?>(null)
    var folder by mutableStateOf<FolderEntry?>(null)
    /**
     * The home screen as it was when [folder] opened (wallpaper copy under the launcher's own
     * window, at a reduced size), for the panel's frosted or glass effect; null when there is
     * none (translucent mode, or no wallpaper copy).
     */
    var folderBackdrop by mutableStateOf<ImageBitmap?>(null)
    /** The open folder's tile on the home screen (window px): the panel grows out of it. */
    var folderFrom: androidx.compose.ui.geometry.Rect? = null
    /** The app being dragged inside the open folder (drawn as a gap), or 0. */
    var folderDragId by mutableLongStateOf(0L)
    /** The open folder's visible page of apps, in window coordinates, and its page number. */
    val folderGrid = android.graphics.RectF()
    var folderPage = 0
    var snack by mutableStateOf<Snack?>(null)
    var dragging by mutableStateOf(false)
    var editMode by mutableStateOf(false)
    /** Icon size on the home screen, in dp; an open folder shows its apps at the same size. */
    var homeIconDp by mutableFloatStateOf(56f)
    /** "Organize apps by category" is on (home screen only mode). */
    var categoryMode by mutableStateOf(false)
    /** Edit mode: the zoomed page's top edge (window px) and its zoom, for the bar above it. */
    var editPageTop by mutableIntStateOf(0)
    var editScale by mutableFloatStateOf(1f)
    /** Edit mode: the page whose delete is waiting for confirmation, or null. */
    var confirmDeletePage by mutableStateOf<Int?>(null)
    /** Edit mode: Categorize apps is waiting for confirmation. */
    var confirmCategorize by mutableStateOf(false)
    var picker by mutableStateOf<WidgetPickerState?>(null)
    var editApp by mutableStateOf<EditAppState?>(null)
    var currentPage by mutableIntStateOf(0)
    var pageCount by mutableIntStateOf(1)

    /** True while something modal (menu, folder, widget picker, a dialog) covers the home screen. */
    val isModal get() = menu != null || folder != null || confirmDeletePage != null || confirmCategorize || picker != null || editApp != null
}
