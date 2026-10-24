package io.github.jacomartins.canyonui.data

import android.content.Context
import androidx.core.content.edit
import io.github.jacomartins.canyonui.model.AppKey
import io.github.jacomartins.canyonui.model.CategoryOrder
import org.json.JSONObject
import java.time.LocalDate

/**
 * How often each app is opened from the launcher, per day for the last [WINDOW_DAYS] days, kept
 * on this phone only. Category folders use it to bring the apps used most to the front
 * (see [CategoryOrder]).
 */
class AppUsage(context: Context) {
    private val prefs = context.getSharedPreferences("app_usage", Context.MODE_PRIVATE)

    // key -> (day -> launches)
    private val counts = HashMap<String, HashMap<Long, Int>>()
    val upSince = HashMap<String, Long>()
    val known = HashSet<String>()

    init {
        try {
            val json = JSONObject(prefs.getString(KEY, "{}")!!)
            json.optJSONObject("counts")?.let { c ->
                for (k in c.keys()) {
                    val days = c.getJSONObject(k)
                    counts[k] = HashMap<Long, Int>().also { m -> for (d in days.keys()) m[d.toLong()] = days.getInt(d) }
                }
            }
            json.optJSONObject("upSince")?.let { u -> for (k in u.keys()) upSince[k] = u.getLong(k) }
            json.optJSONArray("known")?.let { a -> for (i in 0 until a.length()) known += a.getString(i) }
        } catch (e: Exception) {
            counts.clear(); upSince.clear(); known.clear()
        }
    }

    fun today(): Long = LocalDate.now().toEpochDay()

    /** One launch of [key] today. */
    fun record(key: AppKey) {
        val day = today()
        val m = counts.getOrPut(key.toString()) { HashMap() }
        m[day] = (m[day] ?: 0) + 1
        save()
    }

    /** Launches of [key] in the last [WINDOW_DAYS] days. */
    fun score(key: String): Int {
        val from = today() - WINDOW_DAYS
        return counts[key]?.entries?.sumOf { (d, n) -> if (d > from) n else 0 } ?: 0
    }

    fun save() {
        val from = today() - WINDOW_DAYS
        val c = JSONObject()
        for ((k, m) in counts) {
            val days = JSONObject()
            for ((d, n) in m) if (d > from) days.put(d.toString(), n)
            if (days.length() > 0) c.put(k, days)
        }
        val json = JSONObject()
            .put("counts", c)
            .put("upSince", JSONObject(upSince.mapValues { it.value }))
            .put("known", org.json.JSONArray(known.toList()))
        prefs.edit { putString(KEY, json.toString()) }
    }

    private companion object {
        const val KEY = "state"
        const val WINDOW_DAYS = 28
    }
}
