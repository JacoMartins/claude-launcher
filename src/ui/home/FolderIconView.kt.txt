package io.github.jacomartins.canyonui.ui.home

import android.annotation.SuppressLint
import android.content.Context
import android.graphics.Canvas
import android.graphics.Paint
import android.graphics.Rect
import android.graphics.RectF
import android.text.TextPaint
import android.text.TextUtils
import android.util.TypedValue
import android.view.MotionEvent
import android.view.View
import android.view.animation.DecelerateInterpolator
import io.github.jacomartins.canyonui.data.AppEntry
import io.github.jacomartins.canyonui.data.FolderEntry
import io.github.jacomartins.canyonui.data.IconShape
import io.github.jacomartins.canyonui.ui.design.DepthRingPainter
import io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop
import android.graphics.Path
import androidx.core.graphics.drawable.toDrawable
import io.github.jacomartins.canyonui.util.Motion

/**
 * A folder on the home screen or dock.
 * - Normal (1x1): an icon-sized rounded square with a 3x3 preview of its apps; a tap opens it.
 * - Large (2x2): up to nine apps that can be tapped directly. From nine apps on, the last cell
 *   is a small stack that opens the folder (as does a tap on the name): the ninth app in front
 *   and, behind it, the tenth app's icon, or a placeholder in the icon shape when there is none.
 * App icons are drawn as they are (no border or shadow); only the folder backdrop is a surface.
 */
@SuppressLint("ViewConstructor")
class FolderIconView(context: Context, private var style: AppIconView.LabelStyle) : View(context) {

    interface Listener {
        fun onOpenFolder(view: FolderIconView, folder: FolderEntry)
        fun onLaunchFromFolder(view: View, app: AppEntry, bounds: Rect)
    }

    var listener: Listener? = null
    var folder: FolderEntry? = null
        private set

    /** Backdrop colour when the folder has no colour of its own (follows the theme). */
    var defaultColor: Int = 0x991C1C1E.toInt()

    /** The folder colour chosen in settings for folders without their own; 0 = [defaultColor]. */
    var settingsColor: Int = 0
        set(value) { field = value; invalidate() }

    /** The icon shape from settings: a normal folder takes it; a large one its corner style. */
    var shape: IconShape = IconShape.SYSTEM
        set(value) { if (field != value) { field = value; pathSize = -1f; invalidate() } }

    /** Dark mode: the border takes its dark-surface values. */
    var darkSurface = true
        set(value) { if (field != value) { field = value; invalidate() } }

    private val ring = DepthRingPainter(resources.displayMetrics.density)
    private val shapePath = Path()
    private val badgePainter = BadgePainter()
    private val badgeRect = Rect()

    /** Notifications waiting for this folder's apps, together (0 = no badge). */
    var badge = 0
        set(value) { if (field != value) { field = value; invalidate() } }
    var badgeNumbers = true
        set(value) { if (field != value) { field = value; invalidate() } }
    private val largePath = Path()
    private var pathSize = -1f
    private var pathLeft = 0f
    private var pathTop = 0f

    private val dm = resources.displayMetrics
    private val gapPx = AppIconView.LABEL_GAP_DP * dm.density
    private val box = RectF()
    private val cell = Rect()
    private val hit = Rect()
    private val bgPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val bitmapPaint = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG)
    private val placeholderPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val placeholderPath = Path()
    private val stackRect = RectF()
    private val textPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
        textSize = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, AppIconView.LABEL_SP, dm)
        textAlign = Paint.Align.CENTER
    }
    private var shownLabel = ""
    private var downX = 0f
    private var downY = 0f
    private val pressInterpolator = DecelerateInterpolator()

    private val isLarge get() = folder?.isLarge == true

    init {
        isClickable = true
        isLongClickable = true
        isFocusable = true
        setOnClickListener { onTap() }
        WallpaperBackdrop.track(this)
        applyStyle()
    }

    fun bind(entry: FolderEntry) {
        folder = entry
        contentDescription = "${entry.title.ifBlank { "Folder" }}, ${entry.apps.size} apps"
        updateLabel()
        invalidate()
    }

    fun useStyle(newStyle: AppIconView.LabelStyle) {
        if (newStyle === style) return
        style = newStyle
        applyStyle()
    }

    fun applyStyle() {
        textPaint.color = style.color
        val r = if (style.shadowColor == 0) 0f else 3f * dm.density
        textPaint.setShadowLayer(r, 0f, 1f * dm.density, style.shadowColor)
        updateLabel()
        invalidate()
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) = updateLabel()

    private fun updateLabel() {
        val avail = (width - 4 * dm.density).coerceAtLeast(0f)
        val title = folder?.title.orEmpty()
        shownLabel = if (avail <= 0f) title else TextUtils.ellipsize(title, textPaint, avail, TextUtils.TruncateAt.END).toString()
    }

    /** The backdrop area in view coordinates, also used for the drag image. */
    fun backdropBounds(out: RectF): RectF {
        val showLabel = style.showLabel && folder?.title?.isNotBlank() == true
        val textH = if (showLabel) textPaint.fontSpacing + gapPx else 0f
        if (isLarge) {
            val pad = LARGE_MARGIN_DP * dm.density
            val size = minOf(width - 2 * pad, height - textH - 2 * pad)
            val left = (width - size) / 2f
            val top = (height - textH - size) / 2f
            out.set(left, top, left + size, top + size)
        } else {
            val maxIcon = style.maxIconDp * dm.density
            val size = minOf(maxIcon, width * style.iconCellFraction, height - textH)
            val left = (width - size) / 2f
            val top = (height - size - textH) / 2f
            out.set(left, top, left + size, top + size)
        }
        return out
    }

    override fun onDraw(canvas: Canvas) {
        val f = folder ?: return
        backdropBounds(box)
        // A chosen colour carries its own transparency (from the colour picker); colours saved
        // before the picker existed were opaque and get the usual translucency.
        // A colour the user chose is drawn exactly as chosen, opacity included (opaque pink is
        // opaque); only the theme's default colour thins over the glass or blur.
        val own = if (f.color != 0) f.color else settingsColor
        val custom = own != 0
        bgPaint.color = if (custom) own else defaultColor
        // Frosted and glass: the blurred wallpaper first, then less of the folder's own colour.
        val outlinePath = tileOutline()
        // The glass lens follows the outline's real corners (a circle bends all round).
        val glassRadius = tileRadius()
        // A drag image leaves the wallpaper out: the dragged copy draws it live as it moves.
        val behind = if (snapshotting) WallpaperBackdrop.active.value != null else WallpaperBackdrop.draw(canvas, this, outlinePath, glassRadius)
        if (behind && !custom) {
            // Folders keep less of their tint than other surfaces: clearer glass, less frost.
            bgPaint.alpha = (bgPaint.alpha * WallpaperBackdrop.fillShareFor(bgPaint.color) * FOLDER_TINT).toInt()
        }
        if (isLarge) {
            val radius = box.width() * largeCornerShare()
            canvas.drawRoundRect(box, radius, radius, bgPaint)
            ring.draw(canvas, box, radius, darkSurface)
        } else {
            val path = outline()
            canvas.drawPath(path, bgPaint)
            ring.drawPath(canvas, path, box, darkSurface)
        }

        val apps = f.apps
        val slots = SLOTS
        // Large folders: from nine apps on, the last cell opens the folder, so it is drawn as a
        // stack. Normal folders just preview their first nine apps.
        val stacked = isLarge && apps.size >= slots
        val direct = if (stacked) slots - 1 else minOf(apps.size, slots)
        for (i in 0 until direct) {
            cellRect(i, cell)
            canvas.drawBitmap(apps[i].app.icon, null, cell, bitmapPaint)
        }
        if (stacked) drawStack(canvas, apps[direct], apps.getOrNull(direct + 1))
        // A normal folder carries its apps' badge at its own corner; a large folder's apps
        // are visible, so each shows its own.
        if (isLarge) {
            for (i in 0 until direct) {
                cellRect(i, cell)
                badgePainter.draw(canvas, cell, appBadge(apps[i]), badgeNumbers)
            }
        } else {
            badgeRect.set(box.left.toInt(), box.top.toInt(), box.right.toInt(), box.bottom.toInt())
            badgePainter.draw(canvas, badgeRect, badge, badgeNumbers)
        }
        if (style.showLabel && shownLabel.isNotEmpty()) {
            canvas.drawText(shownLabel, width / 2f, box.bottom + gapPx - textPaint.ascent(), textPaint)
        }
    }

    /** Per-app counts, for a large folder's icons (set with [badge]). */
    var appBadges: Map<String, Int> = emptyMap()
        set(value) { if (field != value) { field = value; invalidate() } }

    private fun appBadge(app: AppEntry) = appBadges[app.app.key.packageName] ?: 0

    /**
     * The last cell of a full large folder: [front], smaller, in front of [back] (the next app)
     * or, without one, a placeholder in the icon shape from settings; offset up and to the end by
     * the same amount in width and height so the pair reads as a stack.
     */
    private fun drawStack(canvas: Canvas, front: AppEntry, back: AppEntry?) {
        cellRect(SLOTS - 1, cell)
        val side = cell.width() * STACK_SCALE
        val shift = cell.width() - side
        val rtl = layoutDirection == LAYOUT_DIRECTION_RTL
        // Behind: towards the top and the end.
        val backLeft = if (rtl) cell.left.toFloat() else cell.right - side
        stackRect.set(backLeft, cell.top.toFloat(), backLeft + side, cell.top + side)
        if (back != null) {
            hit.set(stackRect.left.toInt(), stackRect.top.toInt(), stackRect.right.toInt(), stackRect.bottom.toInt())
            canvas.drawBitmap(back.app.icon, null, hit, bitmapPaint)
        } else {
            placeholderPaint.color = if (darkSurface) 0x59FFFFFF else 0x47000000
            val p = iconMask(side, placeholderPath)
            p.offset(stackRect.left, stackRect.top)
            canvas.drawPath(p, placeholderPaint)
        }
        // In front: towards the bottom and the start.
        val frontLeft = if (rtl) backLeft + shift else backLeft - shift
        hit.set(frontLeft.toInt(), (cell.top + shift).toInt(), (frontLeft + side).toInt(), (cell.top + shift + side).toInt())
        canvas.drawBitmap(front.app.icon, null, hit, bitmapPaint)
    }

    /** About the corner radius of the icon shape, as a share of its side. */
    private fun iconCornerShare() = when (shape) {
        IconShape.CIRCLE -> 0.5f
        IconShape.SQUIRCLE -> 0.38f
        IconShape.ROUNDED -> 0.24f
        IconShape.SQUARE -> 0.1f
        IconShape.SYSTEM -> 0.33f
    }

    /** A large folder's corner radius as a share of its side, by icon shape. */
    private fun largeCornerShare() = when (shape) {
        IconShape.SQUARE -> 0.06f
        IconShape.ROUNDED -> 0.16f
        IconShape.CIRCLE -> 0.24f
        IconShape.SQUIRCLE, IconShape.SYSTEM -> 0.2f
    }

    /** A normal folder's outline: the icon shape (or the phone's icon mask) at [box]. */
    /** Set while a drag image is drawn: everything but the wallpaper effect behind the tile. */
    var snapshotting = false

    /** The tile's outline (large: rounded square; small: the icon shape), in view coordinates. */
    private fun tileOutline(): Path = if (isLarge) {
        val radius = box.width() * largeCornerShare()
        largePath.reset()
        largePath.addRoundRect(box, radius, radius, Path.Direction.CW)
        largePath
    } else {
        outline()
    }

    private fun tileRadius(): Float = if (isLarge) box.width() * largeCornerShare() else box.width() * iconCornerShare()

    /** The tile's outline into [out] (view coordinates) for a drag image; returns its corner radius. */
    fun tileShape(out: Path): Float {
        backdropBounds(box)
        out.set(tileOutline())
        return tileRadius()
    }

    private fun outline(): Path {
        val size = box.width()
        if (size == pathSize && box.left == pathLeft && box.top == pathTop) return shapePath
        pathSize = size; pathLeft = box.left; pathTop = box.top
        val p = iconMask(size, shapePath)
        p.offset(box.left, box.top)
        return p
    }

    /** The icon shape from settings at [size] (the phone's own mask for "System"), at 0,0. */
    private fun iconMask(size: Float, out: Path): Path = shape.path(size, out) ?: run {
        val probe = android.graphics.drawable.AdaptiveIconDrawable(
            android.graphics.Color.TRANSPARENT.toDrawable(), android.graphics.Color.TRANSPARENT.toDrawable(),
        )
        probe.setBounds(0, 0, size.toInt(), size.toInt())
        out.set(probe.iconMask)
        out
    }

    /**
     * Cell [i] of the 3x3 inner grid, inset so icons have air around them. A large folder is
     * inset a little more at the top and bottom: its rounded corners otherwise make the space
     * above and below the icons look smaller than at the sides.
     */
    private fun cellRect(i: Int, out: Rect) {
        val insetX = box.width() * if (isLarge) 0.045f else 0.14f
        val insetY = box.height() * if (isLarge) 0.065f else 0.14f
        val stepX = (box.width() - 2 * insetX) / 3f
        val stepY = (box.height() - 2 * insetY) / 3f
        val icon = minOf(stepX, stepY) * if (isLarge) 0.86f else 0.88f
        val col = if (layoutDirection == LAYOUT_DIRECTION_RTL) 2 - i % 3 else i % 3
        val l = box.left + insetX + col * stepX + (stepX - icon) / 2f
        val t = box.top + insetY + (i / 3) * stepY + (stepY - icon) / 2f
        out.set(l.toInt(), t.toInt(), (l + icon).toInt(), (t + icon).toInt())
    }

    @SuppressLint("ClickableViewAccessibility") // Taps still go through performClick().
    override fun onTouchEvent(event: MotionEvent): Boolean {
        if (event.actionMasked == MotionEvent.ACTION_DOWN) {
            downX = event.x
            downY = event.y
        }
        return super.onTouchEvent(event)
    }

    private fun onTap() {
        val f = folder ?: return
        if (isLarge) {
            // With nine or more apps the last cell opens the folder, so a full folder can
            // always be opened (to edit it) from the folder itself.
            for (i in 0 until minOf(f.apps.size, if (f.apps.size >= 9) 8 else 9)) {
                cellRect(i, cell)
                if (cell.contains(downX.toInt(), downY.toInt())) {
                    val loc = IntArray(2).also { getLocationOnScreen(it) }
                    hit.set(cell)
                    hit.offset(loc[0], loc[1])
                    listener?.onLaunchFromFolder(this, f.apps[i], hit)
                    return
                }
            }
        }
        listener?.onOpenFolder(this, f)
    }

    private companion object {
        const val SLOTS = 9

        /** Share of the usual over-blur tint a folder keeps (less frosted than panels). */
        const val FOLDER_TINT = 0.6f

        /** Gap between a large folder's backdrop and its cells, per side. */
        const val LARGE_MARGIN_DP = 8f

        /** Size of each icon in the last cell's stack, as a share of the cell. */
        const val STACK_SCALE = 0.78f
    }

    override fun setPressed(pressed: Boolean) {
        if (pressed == isPressed) return
        super.setPressed(pressed)
        // Large folders hold tappable apps; scaling the whole block would feel wrong.
        val target = if (pressed && !isLarge) Motion.PRESS_SCALE else 1f
        if (Motion.enabled()) {
            animate().scaleX(target).scaleY(target).setDuration(Motion.PRESS_DURATION_MS).setInterpolator(pressInterpolator).start()
        } else {
            scaleX = target
            scaleY = target
        }
    }
}
