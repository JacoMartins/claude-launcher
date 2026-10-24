package io.github.jacomartins.canyonui.search

import android.content.Context
import androidx.core.content.edit
import org.json.JSONObject
import java.time.LocalDate
import kotlin.math.pow

/**
 * Which search results the user picks, kept on this phone only, so the ones used most come first:
 * messaging Daniele on WhatsApp above calling her when that is what is usually done, the people
 * contacted most above others with the same name. Recent use counts more ([HALF_LIFE_DAYS]).
 */
class ActionStats(context: Context) {
    private val prefs = context.applicationContext.getSharedPreferences("search_actions", Context.MODE_PRIVATE)

    // key -> (uses, last day used)
    private val stats = HashMap<String, Pair<Int, Long>>()

    init {
        try {
            val json = JSONObject(prefs.getString(KEY, "{}")!!)
            for (k in json.keys()) {
                val a = json.getJSONArray(k)
                stats[k] = a.getInt(0) to a.getLong(1)
            }
        } catch (e: Exception) {
            stats.clear()
        }
    }

    @Synchronized
    fun record(key: String) {
        val today = LocalDate.now().toEpochDay()
        val (n, _) = stats[key] ?: (0 to today)
        stats[key] = (n + 1) to today
        // Only the most used are kept.
        if (stats.size > MAX_KEYS) {
            stats.entries.sortedBy { score(it.key) }.take(stats.size - MAX_KEYS).forEach { stats.remove(it.key) }
        }
        val json = JSONObject()
        for ((k, v) in stats) json.put(k, org.json.JSONArray().put(v.first).put(v.second))
        prefs.edit { putString(KEY, json.toString()) }
    }

    /** Uses of [key], each counting less the older it is. 0 when never used. */
    @Synchronized
    fun score(key: String): Double {
        val (n, last) = stats[key] ?: return 0.0
        val age = (LocalDate.now().toEpochDay() - last).coerceAtLeast(0)
        return n * 0.5.pow(age / HALF_LIFE_DAYS)
    }

    companion object {
        private const val KEY = "stats"
        private const val MAX_KEYS = 400
        private const val HALF_LIFE_DAYS = 14.0

        /** Any action on a person also counts for the person (who comes first among namesakes). */
        fun person(lookupKey: String) = "person:$lookupKey"
    }
}
