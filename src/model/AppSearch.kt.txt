package io.github.jacomartins.canyonui.model

import java.text.Normalizer

/** Sorting, sectioning and search for app labels. Pure Kotlin. */
object AppSearch {

    private val COMBINING_MARKS = Regex("\\p{Mn}+")

    /** Lower-case, accent-free form used for sorting and matching ("Câmera" -> "camera"). */
    fun normalize(s: String): String =
        COMBINING_MARKS.replace(Normalizer.normalize(s, Normalizer.Form.NFD), "").lowercase().trim()

    /** Fast-scroll section for a label: an upper-case letter, or "#" for digits and symbols. */
    fun sectionOf(label: String): String {
        val first = normalize(label).firstOrNull() ?: return "#"
        return if (first.isLetter()) first.uppercaseChar().toString() else "#"
    }

    /** "#" first, then letters in order, matching how the list is sorted. */
    fun <T> sortByLabel(items: List<T>, label: (T) -> String): List<T> =
        items.sortedWith(compareBy<T>({ sectionOf(label(it)) != "#" }, { normalize(label(it)) }, { label(it) }))

    /**
     * Instant filter. Ranking: label starts with the query, then a word in the label starts with
     * it, then the label contains it. Order within a rank follows [items].
     */
    fun <T> filter(items: List<T>, query: String, label: (T) -> String): List<T> =
        filterNormalized(items, query) { normalize(label(it)) }

    /** [filter] with each item's label already [normalize]d (kept from one key press to the next). */
    fun <T> filterNormalized(items: List<T>, query: String, normalizedLabel: (T) -> String): List<T> {
        val q = normalize(query)
        if (q.isEmpty()) return items
        val prefix = ArrayList<T>()
        val wordStart = ArrayList<T>()
        val contains = ArrayList<T>()
        for (item in items) {
            val l = normalizedLabel(item)
            when {
                l.startsWith(q) -> prefix += item
                hasWordStartingWith(l, q) -> wordStart += item
                l.contains(q) -> contains += item
            }
        }
        return prefix + wordStart + contains
    }

    /**
     * How well [label] matches [normalizedQuery] (already [normalize]d): 0 the label starts with
     * it, 1 a word in it does, 2 it contains it, -1 no match. The same ranks as [filter].
     */
    fun rank(label: String, normalizedQuery: String): Int {
        if (normalizedQuery.isEmpty()) return -1
        val l = normalize(label)
        return when {
            l.startsWith(normalizedQuery) -> 0
            hasWordStartingWith(l, normalizedQuery) -> 1
            l.contains(normalizedQuery) -> 2
            else -> -1
        }
    }

    private fun hasWordStartingWith(label: String, q: String): Boolean {
        var i = label.indexOf(q)
        while (i > 0) {
            if (!label[i - 1].isLetterOrDigit()) return true
            i = label.indexOf(q, i + 1)
        }
        return false
    }

    /** Ordered distinct sections with the index of their first item in [sortedLabels]. */
    fun sectionIndex(sortedLabels: List<String>): List<Pair<String, Int>> {
        val out = ArrayList<Pair<String, Int>>()
        var last: String? = null
        sortedLabels.forEachIndexed { i, l ->
            val s = sectionOf(l)
            if (s != last) {
                out += s to i
                last = s
            }
        }
        return out
    }
}
