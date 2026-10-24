package io.github.jacomartins.canyonui.search

/** What a search text looks like, for the hand-off results (call, email, open link). Pure Kotlin. */
object QueryKind {

    private val PHONE = Regex("""^\+?[0-9(][0-9 ()./-]+[0-9]$""")
    private val EMAIL = Regex("""^[^\s@]+@[^\s@]+\.[A-Za-z]{2,}$""")
    private val URL = Regex("""^(https?://)?([A-Za-z0-9-]+\.)+[A-Za-z]{2,}(:[0-9]+)?(/\S*)?$""", RegexOption.IGNORE_CASE)

    /** A phone number to dial: digits with the usual separators, at least 3 digits. */
    fun phone(q: String): String? {
        val t = q.trim()
        if (!PHONE.matches(t) || t.count { it.isDigit() } < 3) return null
        // A sum such as "12-4" is a calculation, not a number.
        if (Calc.evaluate(t) != null && !t.startsWith("+") && ' ' !in t && '(' !in t) return null
        return t
    }

    // Street type first ("Rua Augusta, 1500", "Av. Paulista 1000", "Street ..."), with a number.
    private val STREET_FIRST = Regex(
        """^(?:rua|r\.|avenida|av\.?|alameda|al\.|travessa|tv\.|estrada|estr\.|rodovia|rod\.|pra[cç]a|p[cç]a\.?|largo|via|viela|""" +
            """street|st\.|avenue|ave\.?|road|rd\.|boulevard|blvd\.?|lane|ln\.|drive|dr\.|calle|carrera)\s+\S.*?\d+.*$""",
        RegexOption.IGNORE_CASE,
    )

    // Number first ("1600 Amphitheatre Parkway", "221B Baker Street").
    private val NUMBER_FIRST = Regex(
        """^\d+[a-z]?\s+(?:[\p{L}'.-]+\s+){0,4}(?:street|st\.?|avenue|ave\.?|road|rd\.?|boulevard|blvd\.?|lane|ln\.?|drive|dr\.?|way|parkway|pkwy|court|ct\.?|place|pl\.?|square|sq\.?|terrace|highway|hwy)\b.*$""",
        RegexOption.IGNORE_CASE,
    )

    // A Brazilian CEP (01310-100) or a US ZIP after a comma ("..., NY 10001").
    private val POSTCODE = Regex("""(?:^|\s|,)(?:\d{5}-\d{3}|[A-Z]{2}\s+\d{5}(?:-\d{4})?)(?:\s|,|$)""")

    /** Text that looks like a street address, for "Open in Maps". Rough by nature. */
    fun address(q: String): String? {
        val t = q.trim()
        if (t.length < 6 || t.length > 160 || '@' in t || "://" in t) return null
        return t.takeIf { STREET_FIRST.matches(it) || NUMBER_FIRST.matches(it) || POSTCODE.containsMatchIn(it) }
    }

    fun email(q: String): String? = q.trim().takeIf { EMAIL.matches(it) }

    /** A web address to open, with "https://" added when missing. */
    fun url(q: String): String? {
        val t = q.trim()
        if (' ' in t || '@' in t || !URL.matches(t)) return null
        // "2.5" or "v1.2" are not addresses: the last label must be letters.
        val host = t.substringAfter("://").substringBefore('/').substringBefore(':')
        if (host.substringAfterLast('.').any { !it.isLetter() }) return null
        return if (t.startsWith("http://", true) || t.startsWith("https://", true)) t else "https://$t"
    }
}
