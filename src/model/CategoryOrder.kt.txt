package io.github.jacomartins.canyonui.model

/**
 * The order of apps inside category folders: alphabetical to begin with, then the apps opened
 * most come first. The order is meant to stay familiar, so an app only moves up after it has
 * deserved a higher place for [HOLD_DAYS] days in a row; it then swaps with the app in that place
 * (whoever put it there). New apps take their place at once. Pure Kotlin, for unit tests.
 */
object CategoryOrder {
    const val HOLD_DAYS = 5

    /**
     * @param current the folder's apps in their present order (keys).
     * @param label each app's name, for the alphabetical order.
     * @param score how much each app was used lately (missing = 0).
     * @param upSince per app, the day it first deserved a higher place; updated in place.
     * @param known apps already ordered before (others are new and placed at once).
     * @param today the current day (days since the epoch).
     */
    fun arrange(
        current: List<String>,
        label: (String) -> String,
        score: (String) -> Int,
        upSince: MutableMap<String, Long>,
        known: Set<String>,
        today: Long,
        compare: Comparator<String> = String.CASE_INSENSITIVE_ORDER,
    ): List<String> {
        val desired = current.sortedWith(compareByDescending<String> { score(it) }.thenBy(compare) { label(it) })
        val result = current.filter { it in known }.toMutableList()
        // New apps go straight to where they belong among the rest.
        for (k in desired) if (k !in known) {
            val at = desired.indexOf(k).coerceAtMost(result.size)
            result.add(at, k)
        }
        for ((d, k) in desired.withIndex()) {
            val cur = result.indexOf(k)
            if (d < cur) {
                val since = upSince.getOrPut(k) { today }
                if (today - since >= HOLD_DAYS) {
                    result[cur] = result[d]
                    result[d] = k
                    upSince.remove(k)
                }
            } else {
                upSince.remove(k)
            }
        }
        upSince.keys.retainAll(current.toSet())
        return result
    }
}
