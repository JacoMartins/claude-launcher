package io.github.jacomartins.canyonui.model

/**
 * Pure helpers for the icon pack `appfilter.xml` format (ADW / Nova / Apex convention):
 *
 * ```
 * <item component="ComponentInfo{com.app/com.app.MainActivity}" drawable="app_icon" />
 * <calendar component="ComponentInfo{com.cal/com.cal.Main}" prefix="calendar_" />
 * ```
 */
object AppFilter {

    /**
     * Parses `ComponentInfo{pkg/cls}` into package and class. A class starting with "." is
     * relative to the package. Returns null for malformed or placeholder entries.
     */
    fun parseComponent(value: String?): Pair<String, String>? {
        val v = value?.trim() ?: return null
        val start = v.indexOf('{')
        val end = v.lastIndexOf('}')
        if (!v.startsWith("ComponentInfo") || start < 0 || end <= start) return null
        val inner = v.substring(start + 1, end).trim()
        val slash = inner.indexOf('/')
        if (slash <= 0 || slash == inner.length - 1) return null
        val pkg = inner.substring(0, slash).trim()
        var cls = inner.substring(slash + 1).trim()
        if (cls.startsWith(".")) cls = pkg + cls
        if (pkg.isEmpty() || cls.isEmpty() || pkg.contains(' ') || cls.contains(' ')) return null
        return pkg to cls
    }

    /** Calendar icons are named prefix + day of month, e.g. "calendar_31". */
    fun calendarDrawable(prefix: String, dayOfMonth: Int): String = prefix + dayOfMonth

    private val WEEKDAYS = arrayOf("sun", "mon", "tue", "wed", "thu", "fri", "sat")

    /**
     * Optional weekday variant, e.g. "calendar_31_fri". The appfilter convention only switches
     * by day of month; packs that also want the weekday on the icon (OneDepth) ship these, and
     * this launcher tries them first. [dayOfWeek] uses java.util.Calendar numbering (Sunday = 1).
     */
    fun calendarWeekdayDrawable(prefix: String, dayOfMonth: Int, dayOfWeek: Int): String =
        "$prefix${dayOfMonth}_${WEEKDAYS[(dayOfWeek - 1).coerceIn(0, 6)]}"

    /**
     * Lookup rule shared by plain and calendar entries: the exact activity first, then any entry
     * for the same package (apps sometimes rename their launcher activity).
     */
    fun <T> lookup(
        packageName: String,
        className: String,
        byComponent: Map<Pair<String, String>, T>,
        byPackage: Map<String, T>,
    ): T? = byComponent[packageName to className] ?: byPackage[packageName]
}
