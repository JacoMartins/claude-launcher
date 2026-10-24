package io.github.jacomartins.canyonui.model

/** First-run layout: a clean dock and one tidy row on the first page. Pure Kotlin. */
object DefaultLayoutPlanner {

    data class Plan(val dock: List<AppKey>, val firstPageRow: List<AppKey>)

    /**
     * [dockCandidates] and [pageCandidates] are in priority order; null means "no app for this
     * role on this device". Duplicates are dropped, the dock is filled first, and the page row
     * gets at most one row of apps that are not already in the dock.
     */
    fun plan(
        dockCandidates: List<AppKey?>,
        pageCandidates: List<AppKey?>,
        dockSlots: Int,
        columns: Int,
    ): Plan {
        val used = LinkedHashSet<AppKey>()
        val dock = dockCandidates.filterNotNull().filter { used.add(it) }.take(dockSlots)
        used.retainAll(dock.toSet())
        val row = pageCandidates.filterNotNull().filter { used.add(it) }.take(columns)
        return Plan(dock, row)
    }

    /** Dock cells for [count] apps in [slots], centred (as far as an integer offset allows). */
    fun dockPositions(count: Int, slots: Int): List<Int> {
        if (count >= slots) return List(slots) { it }
        val start = (slots - count) / 2
        return List(count) { start + it }
    }
}
