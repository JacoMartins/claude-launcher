package io.github.jacomartins.canyonui.model

/** Where a home item lives. */
object Container {
    const val DESKTOP = 0
    const val DOCK = 1

    /** Inside a folder: the item's parent is the folder's id, its x is its rank. */
    const val FOLDER = 2

    /** Inside a widget stack: the widget's parent is the stack's id, its x is its rank. */
    const val STACK = 3
}

object ItemType {
    const val APP = 0
    const val FOLDER = 1
    const val WIDGET = 2

    /** Widgets of one size swiped through in one place. */
    const val STACK = 3

    /** Items that hold others (apps in folders, widgets in stacks). */
    fun isGroup(type: Int) = type == FOLDER || type == STACK
}

/**
 * A stored home item, independent of Room. For apps [key] is the launch target
 * ("pkg/cls#userSerial"); folders, widgets and stacks have no key (a widget whose provider is
 * gone is shown as a placeholder by the UI, not hidden here).
 */
data class StoredItem(
    val id: Long,
    val container: Int,
    val pageIndex: Int,
    val x: Int,
    val y: Int,
    val spanX: Int,
    val spanY: Int,
    val key: AppKey?,
    val missingSince: Long?,
    val type: Int = ItemType.APP,
    val parentId: Long = 0,
)

/** A launchable activity: package, class and the profile (user serial) it belongs to. */
data class AppKey(val packageName: String, val className: String, val userSerial: Long) {
    override fun toString() = "$packageName/$className#$userSerial"

    companion object {
        fun parse(s: String): AppKey? {
            val slash = s.indexOf('/')
            val hash = s.lastIndexOf('#')
            if (slash <= 0 || hash <= slash + 1) return null
            val serial = s.substring(hash + 1).toLongOrNull() ?: return null
            return AppKey(s.substring(0, slash), s.substring(slash + 1, hash), serial)
        }
    }
}

/** Changes the resolver wants written back to storage. */
sealed interface ItemUpdate {
    val id: Long

    data class MarkMissing(override val id: Long, val since: Long) : ItemUpdate
    data class ClearMissing(override val id: Long) : ItemUpdate
    data class Retarget(override val id: Long, val key: AppKey) : ItemUpdate

    /** [pageIndex] may be past the last existing page; storage then appends pages. */
    data class Move(override val id: Long, val container: Int, val pageIndex: Int, val x: Int, val y: Int) : ItemUpdate
    data class Delete(override val id: Long) : ItemUpdate
}

data class PlacedItem(
    val id: Long,
    val container: Int,
    val pageIndex: Int,
    val x: Int,
    val y: Int,
    val spanX: Int,
    val spanY: Int,
    val key: AppKey?,
    val type: Int = ItemType.APP,
) {
    fun covers(cx: Int, cy: Int) = cx in x until x + spanX && cy in y until y + spanY
}

/**
 * [visible] are top-level items (pages and dock). [children] maps a folder id to its visible
 * apps, or a stack id to its widgets, in rank order.
 */
class ResolvedLayout(
    val visible: List<PlacedItem>,
    val updates: List<ItemUpdate>,
    val children: Map<Long, List<PlacedItem>> = emptyMap(),
)

/**
 * Decides what the home screen shows given what is stored and what is installed.
 *
 * - An item whose exact activity is gone but whose package still has a launchable activity is
 *   retargeted to it (apps sometimes rename their launcher activity in an update).
 * - An item whose package is gone is hidden, not deleted, and remembers when it went missing.
 *   If the app comes back within [ghostRetentionMs] (a reinstall, an SD card being remounted)
 *   the icon returns to its old place. After that it is deleted.
 * - Items that no longer fit (grid made smaller, cell taken by something added meanwhile) move
 *   to the first free cell on their page, then the next pages, then a new page.
 * - Folders: their apps follow the same rules. A folder whose apps are all temporarily missing
 *   is hidden but kept; a folder with no apps left is deleted; apps whose folder no longer
 *   exists move onto the pages. Only 1x1 apps and folders fit in the dock.
 * - Stacks follow the folder rules for widgets: an empty stack is deleted, and widgets whose
 *   stack no longer exists move onto the pages with their own size.
 */
object LayoutResolver {

    const val DEFAULT_GHOST_RETENTION_MS = 7L * 24 * 60 * 60 * 1000

    fun resolve(
        items: List<StoredItem>,
        pageCount: Int,
        columns: Int,
        rows: Int,
        dockSlots: Int,
        installed: Set<AppKey>,
        defaultActivityFor: (packageName: String, userSerial: Long) -> AppKey?,
        now: Long,
        ghostRetentionMs: Long = DEFAULT_GHOST_RETENTION_MS,
    ): ResolvedLayout {
        val updates = ArrayList<ItemUpdate>()
        val present = ArrayList<StoredItem>(items.size)
        val groupIds = items.filter { ItemType.isGroup(it.type) }.mapTo(HashSet()) { it.id }
        val children = HashMap<Long, MutableList<StoredItem>>()
        val liveChildren = HashMap<Long, Int>() // Present or still-remembered apps per folder.

        for (item in items) {
            if (ItemType.isGroup(item.type)) continue
            if (item.type == ItemType.WIDGET) {
                val inStack = item.container == Container.STACK
                when {
                    inStack && item.parentId !in groupIds -> present += item.copy(container = Container.DESKTOP, pageIndex = -1)
                    inStack -> {
                        children.getOrPut(item.parentId) { ArrayList() } += item
                        liveChildren.merge(item.parentId, 1, Int::plus)
                    }
                    else -> present += item
                }
                continue
            }
            val key = item.key
            if (key == null) {
                updates += ItemUpdate.Delete(item.id) // An app row without a target is unusable.
                continue
            }
            val inFolder = item.container == Container.FOLDER
            val orphan = inFolder && item.parentId !in groupIds
            val target = when {
                key in installed -> key
                else -> defaultActivityFor(key.packageName, key.userSerial)
            }
            if (target == null) {
                val expired = item.missingSince != null && now - item.missingSince > ghostRetentionMs
                when {
                    orphan || expired -> updates += ItemUpdate.Delete(item.id)
                    item.missingSince == null -> updates += ItemUpdate.MarkMissing(item.id, now)
                }
                // A remembered app keeps its folder alive until it is deleted.
                if (inFolder && !orphan && !expired) liveChildren.merge(item.parentId, 1, Int::plus)
                continue
            }
            if (target != key) updates += ItemUpdate.Retarget(item.id, target)
            if (item.missingSince != null) updates += ItemUpdate.ClearMissing(item.id)
            val resolved = item.copy(key = target)
            when {
                orphan -> present += resolved.copy(container = Container.DESKTOP, pageIndex = -1, spanX = 1, spanY = 1)
                inFolder -> {
                    children.getOrPut(item.parentId) { ArrayList() } += resolved
                    liveChildren.merge(item.parentId, 1, Int::plus)
                }
                else -> present += resolved
            }
        }

        for (folder in items) {
            if (!ItemType.isGroup(folder.type)) continue
            when {
                (liveChildren[folder.id] ?: 0) == 0 -> updates += ItemUpdate.Delete(folder.id)
                children[folder.id].isNullOrEmpty() -> Unit // Every app is temporarily missing: hide, keep.
                else -> present += folder
            }
        }

        // Items that never went missing keep their cells; restored ghosts yield to them.
        val ordered = present.sortedWith(compareBy<StoredItem>({ it.missingSince != null }, { it.id }))

        val pages = ArrayList<GridOccupancy>()
        fun page(i: Int): GridOccupancy {
            while (pages.size <= i) pages += GridOccupancy(columns, rows)
            return pages[i]
        }
        repeat(maxOf(pageCount, 1)) { page(it) }
        val dock = GridOccupancy(dockSlots, 1)

        val visible = ArrayList<PlacedItem>(ordered.size)
        val displaced = ArrayList<StoredItem>()

        for (item in ordered) {
            val fitsDock = item.container == Container.DOCK && item.spanX == 1 && item.spanY == 1 &&
                (item.type == ItemType.APP || item.type == ItemType.FOLDER) && dock.isFree(item.x, 0)
            val fitsPage = item.container == Container.DESKTOP && item.pageIndex in 0 until pageCount &&
                page(item.pageIndex).isFree(item.x, item.y, item.spanX, item.spanY)
            when {
                fitsDock -> {
                    dock.mark(item.x, 0)
                    visible += item.placed(Container.DOCK, 0, item.x, 0)
                }
                fitsPage -> {
                    page(item.pageIndex).mark(item.x, item.y, item.spanX, item.spanY)
                    visible += item.placed(Container.DESKTOP, item.pageIndex, item.x, item.y)
                }
                else -> displaced += item
            }
        }

        for (item in displaced) {
            // Prefer the item's own page, then pages in order, then a fresh page.
            val start = if (item.container == Container.DESKTOP) item.pageIndex.coerceIn(0, pages.size - 1) else 0
            val order = listOf(start) + pages.indices.filter { it != start }
            var placed = false
            for (p in order) {
                val cell = pages[p].findFirstFree(item.spanX, item.spanY) ?: continue
                pages[p].mark(cell.x, cell.y, item.spanX, item.spanY)
                visible += item.placed(Container.DESKTOP, p, cell.x, cell.y)
                updates += ItemUpdate.Move(item.id, Container.DESKTOP, p, cell.x, cell.y)
                placed = true
                break
            }
            if (!placed) {
                val p = pages.size
                val cell = page(p).findFirstFree(item.spanX, item.spanY)
                if (cell == null) {
                    // Larger than an empty page (only possible after shrinking the grid).
                    updates += ItemUpdate.Delete(item.id)
                } else {
                    pages[p].mark(cell.x, cell.y, item.spanX, item.spanY)
                    visible += item.placed(Container.DESKTOP, p, cell.x, cell.y)
                    updates += ItemUpdate.Move(item.id, Container.DESKTOP, p, cell.x, cell.y)
                }
            }
        }

        val visibleChildren = children.mapValues { (_, list) ->
            list.sortedWith(compareBy({ it.x }, { it.id })).map { it.placed(it.container, 0, it.x, 0) }
        }
        return ResolvedLayout(visible, updates, visibleChildren)
    }

    private fun StoredItem.placed(container: Int, page: Int, x: Int, y: Int) =
        PlacedItem(id, container, page, x, y, spanX, spanY, key, type)

    /**
     * Where to put a new item ([spanX] x [spanY], an app is 1x1): the first free spot on
     * [preferredPage], then on any page, else on a new page (returned index == page count).
     */
    fun findSlotForNewItem(
        visible: List<PlacedItem>,
        pageCount: Int,
        columns: Int,
        rows: Int,
        preferredPage: Int,
        spanX: Int = 1,
        spanY: Int = 1,
    ): Pair<Int, Cell> {
        val pages = List(maxOf(pageCount, 1)) { GridOccupancy(columns, rows) }
        for (v in visible) {
            if (v.container == Container.DESKTOP && v.pageIndex in pages.indices) {
                pages[v.pageIndex].mark(v.x, v.y, v.spanX, v.spanY)
            }
        }
        val start = preferredPage.coerceIn(0, pages.size - 1)
        for (p in listOf(start) + pages.indices.filter { it != start }) {
            pages[p].findFirstFree(spanX, spanY)?.let { return p to it }
        }
        return pages.size to Cell(0, 0)
    }

    data class NewPlacement(val key: AppKey, val pageIndex: Int, val cell: Cell)

    /**
     * Home-screen-only mode: places [missing] apps (in the given order) into free 1x1 cells,
     * starting from the last page that has anything on it and appending pages as needed.
     * Earlier pages are left alone so the user's arrangement is not filled in from the top.
     */
    fun placeMissingApps(
        visible: List<PlacedItem>,
        pageCount: Int,
        columns: Int,
        rows: Int,
        missing: List<AppKey>,
    ): List<NewPlacement> {
        if (missing.isEmpty()) return emptyList()
        val pages = ArrayList<GridOccupancy>()
        repeat(maxOf(pageCount, 1)) { pages += GridOccupancy(columns, rows) }
        var lastUsed = 0
        for (v in visible) {
            if (v.container != Container.DESKTOP) continue
            while (pages.size <= v.pageIndex) pages += GridOccupancy(columns, rows)
            pages[v.pageIndex].mark(v.x, v.y, v.spanX, v.spanY)
            lastUsed = maxOf(lastUsed, v.pageIndex)
        }
        val out = ArrayList<NewPlacement>(missing.size)
        var page = lastUsed
        for (key in missing) {
            var cell: Cell? = null
            while (cell == null) {
                while (pages.size <= page) pages += GridOccupancy(columns, rows)
                cell = pages[page].findFirstFree()
                if (cell == null) page++
            }
            pages[page].mark(cell.x, cell.y)
            out += NewPlacement(key, page, cell)
        }
        return out
    }
}
