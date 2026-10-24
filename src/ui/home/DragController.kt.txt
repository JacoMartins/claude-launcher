package io.github.jacomartins.canyonui.ui.home

import android.annotation.SuppressLint
import android.content.Context
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.Paint
import android.graphics.Rect
import android.view.HapticFeedbackConstants
import android.view.View
import android.view.animation.DecelerateInterpolator
import android.widget.FrameLayout
import io.github.jacomartins.canyonui.data.AppInfo
import io.github.jacomartins.canyonui.data.DragItem
import io.github.jacomartins.canyonui.data.HomeState
import io.github.jacomartins.canyonui.model.Cell
import io.github.jacomartins.canyonui.model.Container
import io.github.jacomartins.canyonui.model.DropAction
import io.github.jacomartins.canyonui.model.DropLogic
import io.github.jacomartins.canyonui.util.Motion
import kotlin.math.abs

/** What is being dragged and how to draw it. */
class DragSource(
    val item: DragItem,
    val type: Int,
    val spanX: Int,
    val spanY: Int,
    val image: Bitmap,
    /** Image size and the finger's offset inside it, in screen pixels. */
    val width: Int,
    val height: Int,
    val touchX: Float,
    val touchY: Float,
    /** Hidden while dragging; null for items coming from the drawer or a folder popup. */
    val view: View?,
    val app: AppInfo?,
    /** Dragged out of this folder's open popup (0 = not from a folder). */
    val fromFolder: Long = 0,
    /**
     * A folder's tile outline in [image]'s pixels: the dragged copy draws the frosted or glass
     * wallpaper inside it live, so the effect follows the wallpaper as it moves.
     */
    val backdrop: android.graphics.Path? = null,
    val backdropRadius: Float = 0f,
)

/** Where a drag ended. */
sealed interface DropResult {
    data class OnGrid(val action: DropAction) : DropResult

    /** A whole page (edit mode) dropped while page [index] is in view. */
    data class OnPage(val index: Int) : DropResult

    /** An app dropped back inside its open folder, at position [index]. */
    data class InFolder(val index: Int) : DropResult
    data object Cancelled : DropResult
}

/** The icon under the finger. Moves by translation only. */
@SuppressLint("ViewConstructor")
class DragView(
    context: Context,
    private val bitmap: Bitmap,
    private val backdrop: android.graphics.Path? = null,
    private val backdropRadius: Float = 0f,
) : View(context) {
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG)
    private val dst = Rect()
    private val scaled = android.graphics.Path()
    private val matrix = android.graphics.Matrix()
    private var radius = 0f

    init {
        importantForAccessibility = IMPORTANT_FOR_ACCESSIBILITY_NO
        // The wallpaper behind it changes as it moves: redrawn whenever it does.
        if (backdrop != null) io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.track(this)
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        val b = backdrop ?: return
        val sx = w / bitmap.width.toFloat()
        matrix.setScale(sx, h / bitmap.height.toFloat())
        b.transform(matrix, scaled)
        radius = backdropRadius * sx
    }

    override fun onDraw(canvas: Canvas) {
        if (backdrop != null) io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.draw(canvas, this, scaled, radius)
        dst.set(0, 0, width, height)
        canvas.drawBitmap(bitmap, null, dst, paint)
    }
}

/**
 * Drives a drag from a long-press to the drop. Hit-tests the dock and the current page; asks [DropLogic] what a release would do; shows the drop hint; ticks a haptic when the
 * target changes; flips pages when the finger rests at a screen edge.
 */
class DragController(
    private val root: FrameLayout,
    private val home: HomeScreenLayout,
    private val callbacks: Callbacks,
) {
    interface Callbacks {
        fun state(): HomeState?
        fun onDragStarted(source: DragSource)
        fun onDropped(source: DragSource, result: DropResult)

        /** The position in the open folder under the point (root coordinates), or null outside it. */
        fun folderTarget(x: Float, y: Float): Int?

        /** The drag left the folder: close it and continue on home. */
        fun leaveFolder()

        /** Where [view]'s visible part (icon, folder backdrop, widget) is now, in root coordinates. */
        fun landingBounds(view: View): Rect?
    }

    // A drag from a folder stays in the folder (reordering) until the finger rests outside it.
    private var inFolder = false
    private var folderIndex = -1
    private var leavePending = false
    private val leave = Runnable {
        leavePending = false
        if (!isDragging || !inFolder) return@Runnable
        inFolder = false
        callbacks.leaveFolder()
        update(lastX, lastY)
    }

    var armed: DragSource? = null
        private set

    /** The dropped item's view while the drag image flies into its cell (kept hidden until then). */
    var landing: View? = null
        private set
    var source: DragSource? = null
        private set
    val isDragging get() = source != null

    private var dragView: DragView? = null
    private val pt = FloatArray(2)
    private val cellOut = IntArray(2)
    private var lastAction: DropAction? = null
    private var lastHintCell: CellLayout? = null
    private var lastX = 0f
    private var lastY = 0f
    // What the last drop decision was based on; it is only recomputed when one of these changes.
    private var lastKey = Long.MIN_VALUE
    private val edge = 28 * root.resources.displayMetrics.density
    private var pageFlipPending = false
    private val decelerate = DecelerateInterpolator()

    private val pageFlip = Runnable {
        pageFlipPending = false
        if (!isDragging) return@Runnable
        val ws = home.workspace
        val rtl = ws.layoutDirection == View.LAYOUT_DIRECTION_RTL
        val forward = (lastX > root.width - edge) != rtl
        val next = ws.currentPage + if (forward) 1 else -1
        if (next in 0 until ws.childCount) {
            ws.snapToPage(next)
            root.performHapticFeedback(HapticFeedbackConstants.CLOCK_TICK)
            update(lastX, lastY)
            scheduleFlipIfAtEdge()
        }
    }

    /** Called on long-press: a drag starts if the finger then moves. */
    fun arm(source: DragSource) {
        armed = source
    }

    fun disarm() {
        armed = null
    }

    fun begin(x: Float, y: Float) {
        val s = armed ?: return
        armed = null
        source = s
        inFolder = s.fromFolder != 0L
        folderIndex = -1
        callbacks.onDragStarted(s)
        s.view?.visibility = View.INVISIBLE
        val dv = DragView(root.context, s.image, s.backdrop, s.backdropRadius)
        root.addView(dv, FrameLayout.LayoutParams(s.width, s.height))
        dragView = dv
        dv.pivotX = s.touchX
        dv.pivotY = s.touchY
        if (Motion.enabled()) {
            dv.scaleX = 1f; dv.scaleY = 1f
            dv.animate().scaleX(DRAG_SCALE).scaleY(DRAG_SCALE).setDuration(120).start()
        }
        update(x, y)
    }

    fun move(x: Float, y: Float) {
        if (!isDragging) return
        update(x, y)
        scheduleFlipIfAtEdge()
    }

    fun drop(x: Float, y: Float) {
        val s = source ?: return
        update(x, y)
        val result = when {
            inFolder -> if (folderIndex >= 0) DropResult.InFolder(folderIndex) else DropResult.Cancelled
            s.item is DragItem.Page -> DropResult.OnPage(home.workspace.currentPage)
            lastAction != null && lastAction != DropAction.Reject -> DropResult.OnGrid(lastAction!!)
            else -> DropResult.Cancelled
        }
        finish(s, result)
    }

    fun cancel() {
        val s = source ?: return
        finish(s, DropResult.Cancelled)
    }

    private fun finish(s: DragSource, result: DropResult) {
        root.removeCallbacks(pageFlip)
        root.removeCallbacks(leave)
        leavePending = false
        inFolder = false
        pageFlipPending = false
        lastHintCell?.clearDropHint()
        lastHintCell = null
        val dv = dragView
        dragView = null
        source = null
        lastAction = null
        lastKey = Long.MIN_VALUE
        val lands = dv != null && Motion.enabled() && s.view != null && landsInView(result)
        root.performHapticFeedback(HapticFeedbackConstants.VIRTUAL_KEY)
        callbacks.onDropped(s, result)
        if (dv == null) return
        if (lands) {
            // The image flies from the finger to where the item now is; the real view shows when
            // it lands. Measured a frame later, once the item has been laid out in its new cell.
            val view = s.view!!
            landing = view
            view.visibility = View.INVISIBLE
            dv.animate().cancel()
            root.post {
                val to = callbacks.landingBounds(view)
                val done = {
                    if (landing === view) landing = null
                    if (!isDragging) view.visibility = View.VISIBLE
                    root.removeView(dv)
                }
                if (to == null || to.isEmpty) { done(); return@post }
                // Keep where the image is on screen while its pivot moves to the top-left.
                val left = dv.translationX + dv.pivotX * (1f - dv.scaleX)
                val top = dv.translationY + dv.pivotY * (1f - dv.scaleY)
                dv.pivotX = 0f
                dv.pivotY = 0f
                dv.translationX = left
                dv.translationY = top
                val scale = to.width() / dv.width.coerceAtLeast(1).toFloat()
                dv.animate().translationX(to.left.toFloat()).translationY(to.top.toFloat()).scaleX(scale).scaleY(scale)
                    .setDuration(LAND_MS).setInterpolator(decelerate)
                    .withEndAction(done).start()
            }
        } else if (Motion.enabled()) {
            dv.animate().alpha(0f).scaleX(1f).scaleY(1f).setDuration(140).setInterpolator(decelerate)
                .withEndAction { root.removeView(dv) }.start()
        } else {
            root.removeView(dv)
        }
    }

    /** A move that ends on the page in view or the dock, where the item can be seen landing. */
    private fun landsInView(result: DropResult): Boolean {
        val a = (result as? DropResult.OnGrid)?.action as? DropAction.Move ?: return result is DropResult.Cancelled
        return a.container == Container.DOCK || a.pageIndex == home.workspace.currentPage
    }

    /** Over a folder (or an app that would become one) the image shrinks, as if going in. */
    private fun shrinkForFolder(into: Boolean) {
        val dv = dragView ?: return
        val target = if (into) FOLDER_SCALE else DRAG_SCALE
        if (dv.scaleX == target) return
        if (Motion.enabled()) {
            dv.animate().scaleX(target).scaleY(target).setDuration(150).setInterpolator(decelerate).start()
        } else {
            dv.scaleX = target
            dv.scaleY = target
        }
    }

    private fun update(x: Float, y: Float) {
        val s = source ?: return
        lastX = x
        lastY = y
        dragView?.let {
            it.translationX = x - s.touchX
            it.translationY = y - s.touchY
        }

        // A page being moved only flips pages at the screen edges; there is no cell to aim at.
        if (s.item is DragItem.Page) return
        if (inFolder) {
            val idx = callbacks.folderTarget(x, y)
            if (idx != null) {
                folderIndex = idx
                if (leavePending) {
                    root.removeCallbacks(leave)
                    leavePending = false
                }
            } else if (!leavePending) {
                leavePending = true
                root.postDelayed(leave, LEAVE_FOLDER_MS)
            }
            return
        }
        val st = callbacks.state() ?: return
        // Dock first, then the page in view.
        val overDock = isOver(home.hotseat, x, y)
        val cellLayout: CellLayout = if (overDock) home.hotseat else home.workspace.pageAt(home.workspace.currentPage) ?: return
        val container = if (overDock) Container.DOCK else Container.DESKTOP
        val pageIndex = if (overDock) 0 else home.workspace.currentPage

        // Finger cell and whether the finger is near its centre (folder intent).
        pt[0] = x; pt[1] = y
        ViewMap.rootToLocal(root, cellLayout, pt)
        val fx = pt[0]
        val fy = pt[1]
        if (!cellLayout.cellAt(fx, fy, cellOut)) {
            lastHintCell?.clearDropHint()
            lastHintCell = null
            lastAction = DropAction.Reject
            return
        }
        val finger = Cell(cellOut[0], cellOut[1])
        val cw = cellLayout.cellWidth.toFloat()
        val ch = cellLayout.cellHeight.toFloat()
        val cx = cellLayout.cellLeft(finger.x) + cw / 2f
        val cy = cellLayout.cellTop(finger.y) + ch / 2f
        val centerHit = abs(fx - cx) < cw * 0.28f && abs(fy - cy) < ch * 0.28f

        // For spans, the top-left cell follows the image's top-left corner.
        val desired = if (s.spanX == 1 && s.spanY == 1) {
            finger
        } else {
            val scale = ViewMap.scaleOf(root, cellLayout)
            Cell(
                (finger.x - ((s.touchX / scale) / cw).toInt()).coerceAtLeast(0),
                (finger.y - ((s.touchY / scale) / ch).toInt()).coerceAtLeast(0),
            )
        }
        val key = ((container.toLong() * 64 + pageIndex) * 64 + finger.x) * 4096L + finger.y * 64L + desired.x * 8L + desired.y +
            (if (centerHit) 1L shl 40 else 0L) + (System.identityHashCode(st).toLong() shl 41)
        if (key == lastKey && lastHintCell === cellLayout) return
        lastKey = key
        val columns = if (overDock) st.dockSlots else st.columns
        val rows = if (overDock) 1 else st.rows
        val draggedId = (s.item as? DragItem.Existing)?.id
        // Where it came from, for a swap (not for apps taken out of a folder).
        val origin = if (s.fromFolder == 0L) draggedId?.let { st.entry(it)?.placed } else null
        val action = DropLogic.decide(
            container, pageIndex, columns, rows, st.placedIn(container, pageIndex),
            desired, finger, centerHit, draggedId, s.type, s.spanX, s.spanY, origin = origin,
        )
        if (action != lastAction) {
            lastAction = action
            shrinkForFolder(action is DropAction.CreateFolder || action is DropAction.AddToFolder)
            if (lastHintCell !== cellLayout) lastHintCell?.clearDropHint()
            lastHintCell = cellLayout
            when (action) {
                is DropAction.Move -> cellLayout.showDropHint(action.x, action.y, s.spanX, s.spanY, folder = false)
                is DropAction.CreateFolder, is DropAction.AddToFolder -> {
                    val id = (action as? DropAction.CreateFolder)?.targetId ?: (action as DropAction.AddToFolder).folderId
                    val t = st.entry(id)?.placed
                    if (t != null) cellLayout.showDropHint(t.x, t.y, t.spanX, t.spanY, folder = t.spanX == 1)
                }
                is DropAction.CreateStack, is DropAction.AddToStack -> {
                    val id = (action as? DropAction.CreateStack)?.targetId ?: (action as DropAction.AddToStack).stackId
                    val t = st.entry(id)?.placed
                    if (t != null) cellLayout.showDropHint(t.x, t.y, t.spanX, t.spanY, folder = false)
                }
                DropAction.Reject -> cellLayout.clearDropHint()
            }
            if (action != DropAction.Reject) root.performHapticFeedback(HapticFeedbackConstants.CLOCK_TICK)
        }
    }

    private fun isOver(v: View, x: Float, y: Float): Boolean {
        if (!v.isShown) return false
        pt[0] = x; pt[1] = y
        ViewMap.rootToLocal(root, v, pt)
        return pt[0] >= 0 && pt[1] >= 0 && pt[0] < v.width && pt[1] < v.height
    }

    private fun scheduleFlipIfAtEdge() {
        if (inFolder) return
        val atEdge = lastX < edge || lastX > root.width - edge
        if (atEdge && !pageFlipPending) {
            pageFlipPending = true
            root.postDelayed(pageFlip, PAGE_FLIP_DELAY_MS)
        } else if (!atEdge && pageFlipPending) {
            root.removeCallbacks(pageFlip)
            pageFlipPending = false
        }
    }

    companion object {
        private const val DRAG_SCALE = 1.08f

        /** The drag image over a folder target. */
        private const val FOLDER_SCALE = 0.8f

        /** The drag image's flight into its cell after a drop. */
        private const val LAND_MS = 260L
        private const val PAGE_FLIP_DELAY_MS = 550L

        /** How long the finger must stay outside an open folder before the app leaves it. */
        private const val LEAVE_FOLDER_MS = 450L
    }
}
