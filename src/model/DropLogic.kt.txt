package io.github.jacomartins.canyonui.model

/** What releasing a dragged item over a page or the dock does. */
sealed interface DropAction {
    /** Placed at ([x], [y]); items in the way move as listed in [pushes] (pushed aside or swapped). */
    data class Move(
        val container: Int,
        val pageIndex: Int,
        val x: Int,
        val y: Int,
        val pushes: List<ItemMove> = emptyList(),
    ) : DropAction

    /** Dropped onto the centre of another app: both go into a new folder at that app's cell. */
    data class CreateFolder(val targetId: Long) : DropAction

    /** Dropped onto the centre of a folder. */
    data class AddToFolder(val folderId: Long) : DropAction

    /** A widget dropped exactly over another widget of the same size: both become a stack. */
    data class CreateStack(val targetId: Long) : DropAction

    /** A widget dropped exactly over a stack of the same size. */
    data class AddToStack(val stackId: Long) : DropAction

    /** Nowhere to put it (page or dock full, or a large folder over the dock). */
    data object Reject : DropAction
}

/** Another item moved to make room for a drop. */
data class ItemMove(val id: Long, val container: Int, val pageIndex: Int, val x: Int, val y: Int)

/** Drop and folder placement rules. Pure Kotlin so they can be unit-tested. */
object DropLogic {

    /**
     * @param items the items already in the target container/page.
     * @param desired the cell the dragged item's top-left is over.
     * @param fingerCell the cell under the finger.
     * @param centerHit whether the finger is near the centre of [fingerCell] (folder intent)
     *   rather than near its edge (placement intent).
     * @param draggedId the dragged item's id, or null for a new item (from the drawer or a folder).
     * @param origin where the dragged item was before the drag (any page or the dock), for a swap;
     *   null for new items and apps taken out of a folder.
     */
    fun decide(
        container: Int,
        pageIndex: Int,
        columns: Int,
        rows: Int,
        items: List<PlacedItem>,
        desired: Cell,
        fingerCell: Cell,
        centerHit: Boolean,
        draggedId: Long?,
        draggedType: Int,
        spanX: Int,
        spanY: Int,
        /** Folders whose apps are chosen automatically: nothing can be dropped into them. */
        locked: Set<Long> = emptySet(),
        origin: PlacedItem? = null,
    ): DropAction {
        if (container == Container.DOCK && (spanX != 1 || spanY != 1 || draggedType == ItemType.WIDGET || draggedType == ItemType.STACK)) {
            return DropAction.Reject
        }
        val others = items.filter { it.id != draggedId }

        if (centerHit && draggedType == ItemType.APP && spanX == 1 && spanY == 1) {
            val occupant = others.firstOrNull { it.covers(fingerCell.x, fingerCell.y) }?.takeIf { it.id !in locked }
            when (occupant?.type) {
                ItemType.APP -> return DropAction.CreateFolder(occupant.id)
                ItemType.FOLDER -> return DropAction.AddToFolder(occupant.id)
            }
        }

        if (draggedType == ItemType.WIDGET) {
            // Stacking: the widget's outline sits exactly on a widget or stack of its size.
            val occupant = others.firstOrNull { it.x == desired.x && it.y == desired.y && it.spanX == spanX && it.spanY == spanY }
            when (occupant?.type) {
                ItemType.WIDGET -> return DropAction.CreateStack(occupant.id)
                ItemType.STACK -> return DropAction.AddToStack(occupant.id)
            }
        }

        val grid = GridOccupancy(columns, rows)
        for (o in others) grid.mark(o.x, o.y, o.spanX, o.spanY)
        val x = desired.x.coerceIn(0, (columns - spanX).coerceAtLeast(0))
        val y = desired.y.coerceIn(0, (rows - spanY).coerceAtLeast(0))
        if (grid.isFree(x, y, spanX, spanY)) return DropAction.Move(container, pageIndex, x, y)

        // Taken: push what is in the way to the nearest free cells, keeping the dropped item
        // where it was aimed.
        val inWay = others.filter { it.overlaps(x, y, spanX, spanY) }
        push(columns, rows, others, inWay, x, y, spanX, spanY)?.let { moves ->
            return DropAction.Move(container, pageIndex, x, y, moves.map { (o, c) -> ItemMove(o.id, container, pageIndex, c.x, c.y) })
        }
        // No room to push: swap with an item of the same size sitting exactly there.
        val only = inWay.singleOrNull()
        if (origin != null && only != null && only.x == x && only.y == y && only.spanX == spanX && only.spanY == spanY &&
            !(origin.container == Container.DOCK && (only.type == ItemType.WIDGET || only.type == ItemType.STACK))
        ) {
            return DropAction.Move(container, pageIndex, x, y, listOf(ItemMove(only.id, origin.container, origin.pageIndex, origin.x, origin.y)))
        }
        val cell = grid.findNearestFree(x, y, spanX, spanY)
        return cell?.let { DropAction.Move(container, pageIndex, it.x, it.y) } ?: DropAction.Reject
    }

    /** New cells for [inWay] once ([x], [y]) is taken, nearest to where each was; null if one has none. */
    private fun push(
        columns: Int,
        rows: Int,
        others: List<PlacedItem>,
        inWay: List<PlacedItem>,
        x: Int,
        y: Int,
        spanX: Int,
        spanY: Int,
    ): List<Pair<PlacedItem, Cell>>? {
        val grid = GridOccupancy(columns, rows)
        for (o in others) if (o !in inWay) grid.mark(o.x, o.y, o.spanX, o.spanY)
        grid.mark(x, y, spanX, spanY)
        return inWay.map { o ->
            val c = grid.findNearestFree(o.x, o.y, o.spanX, o.spanY) ?: return null
            grid.mark(c.x, c.y, o.spanX, o.spanY)
            o to c
        }
    }

    private fun PlacedItem.overlaps(ox: Int, oy: Int, sx: Int, sy: Int) =
        x < ox + sx && ox < x + spanX && y < oy + sy && oy < y + spanY

    /**
     * Where a folder at ([x], [y]) can grow to 2x2 on its page: first the four 2x2 blocks that
     * contain its current cell, then the nearest free 2x2 anywhere on the page. Null if none.
     */
    fun largeFolderSpot(
        folderId: Long,
        x: Int,
        y: Int,
        columns: Int,
        rows: Int,
        items: List<PlacedItem>,
    ): Cell? {
        val grid = GridOccupancy(columns, rows)
        for (o in items) if (o.id != folderId) grid.mark(o.x, o.y, o.spanX, o.spanY)
        for ((ax, ay) in listOf(x to y, x - 1 to y, x to y - 1, x - 1 to y - 1)) {
            if (grid.isFree(ax, ay, 2, 2)) return Cell(ax, ay)
        }
        return grid.findNearestFree(x, y, 2, 2)
    }
}
