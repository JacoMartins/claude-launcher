package io.github.jacomartins.canyonui.search

/**
 * Commands typed into Canyon search ("call ana", "stories on instagram", "timer 5 min"), in
 * English and Portuguese, ignoring case. Pure Kotlin: this only reads the text; [SearchModel]
 * finds the contact or app each command names.
 */
sealed interface Command {
    /** Through which app a call or message goes; null = the usual (phone, or the preferred chat app). */
    enum class Via { WHATSAPP, SMS }

    data class Call(val who: String, val via: Via? = null) : Command
    data class VideoCall(val who: String) : Command
    data class Message(val who: String, val via: Via? = null) : Command
    data class Email(val who: String) : Command

    /** Music or video; [app] is the text after "on", which may or may not name an app. */
    data class Play(val what: String, val app: String?) : Command
    data class Directions(val place: String) : Command
    data class Timer(val seconds: Int) : Command
    data class Alarm(val hour: Int, val minute: Int) : Command

    /** "<what> on <app>": [app] may or may not name an app (the model checks). */
    data class InApp(val what: String, val app: String) : Command
}

object Commands {

    private fun rx(s: String) = Regex(s, RegexOption.IGNORE_CASE)

    // "to", "para", "pra"... between a verb and its object.
    private const val TO = """(?:(?:to|para|pra|pro|com|with)\s+)?"""

    private val WHATSAPP = """(?:whats\s?app|zap|zapzap|wpp|whats)"""
    private val WA_CALL = rx("""^$WHATSAPP\s+(?:call|ligar|ligacao|ligação|chamada)\s+$TO(.+)$""")
    private val VIDEO = rx("""^(?:video\s?call|videochamada|video\s?chamada|chamada de v[ií]deo)\s+$TO(.+)$""")
    private val CALL = rx("""^(?:call|phone|ring|ligar|ligue|liga|telefonar|telefone)\s+$TO(.+)$""")
    private val WA_MESSAGE = rx("""^$WHATSAPP\s+$TO(.+)$""")
    private val SMS = rx("""^(?:sms|text)\s+$TO(.+)$""")
    private val MESSAGE = rx("""^(?:message|msg|chat|mensagem|mandar mensagem|enviar mensagem|conversar|falar)\s+$TO(.+)$""")
    private val EMAIL = rx("""^(?:e-?mail|mail)\s+$TO(.+)$""")
    private val PLAY = rx("""^(?:play|tocar|toca|toque|ouvir|escutar)\s+(.+)$""")
    private val DIRECTIONS = rx(
        """^(?:directions?(?:\s+to)?|navigate(?:\s+to)?|take me to|go to|como chegar(?:\s+(?:em|a|ao|na|no))?|ir (?:para|pra|ao|até|ate)|rota (?:para|pra|até|ate))\s+(.+)$""",
    )
    private val TIMER_FIRST = rx("""^(?:set\s+(?:a\s+)?)?(?:timer|temporizador|cron[oô]metro)(?:\s+(?:for|de|por|of))?\s+(.+)$""")
    private val TIMER_LAST = rx("""^(.+?)\s+(?:timer|temporizador)$""")
    private val ALARM = rx(
        """^(?:set\s+(?:an?\s+)?)?(?:alarm|alarme|despertador|wake me(?:\s+up)?|me acorde)(?:\s+(?:for|at|para|pra|[aà]s|as))?\s+(\d{1,2})(?:\s*[:h.]\s*(\d{2}))?\s*(am|pm|h)?$""",
    )
    private val OPEN = rx("""^(?:open|launch|abrir|abre|abra)\s+(.+)$""")

    // "<what> on <app>": the last " on " (or Portuguese "no", "na", "em").
    private val ON = rx("""^(.+)\s+(?:on|in|no|na|em|at)\s+(.+)$""")

    private val DURATION_PART = rx(
        """(\d+(?:[.,]\d+)?)\s*(hours|hour|hrs|hr|horas|hora|h|minutes|minute|minutos|minuto|mins|min|m|seconds|second|segundos|segundo|secs|sec|seg|s)?""",
    )

    fun parse(text: String): Command? {
        val q = text.trim().replace(Regex("\\s+"), " ")
        if (q.isEmpty()) return null
        fun Regex.arg(): String? = matchEntire(q)?.groupValues?.get(1)?.trim()?.takeIf { it.isNotEmpty() }

        WA_CALL.arg()?.let { return Command.Call(it, Command.Via.WHATSAPP) }
        VIDEO.arg()?.let { return Command.VideoCall(it) }
        CALL.arg()?.let { return Command.Call(it) }
        SMS.arg()?.let { return Command.Message(it, Command.Via.SMS) }
        MESSAGE.arg()?.let { return Command.Message(it) }
        EMAIL.arg()?.let { return Command.Email(it) }
        ALARM.matchEntire(q)?.let { m -> alarm(m)?.let { return it } }
        TIMER_FIRST.arg()?.let { s -> duration(s)?.let { return Command.Timer(it) } }
        TIMER_LAST.arg()?.let { s -> duration(s)?.let { return Command.Timer(it) } }
        DIRECTIONS.arg()?.let { return Command.Directions(it) }
        PLAY.arg()?.let { what ->
            val on = ON.matchEntire(what)
            return if (on != null) Command.Play(on.groupValues[1].trim(), on.groupValues[2].trim()) else Command.Play(what, null)
        }
        // "zap ana" after the verbs above, so "whatsapp call ana" is not a message to "call ana".
        WA_MESSAGE.arg()?.let { return Command.Message(it, Command.Via.WHATSAPP) }
        ON.matchEntire(q)?.let { m -> return Command.InApp(m.groupValues[1].trim(), m.groupValues[2].trim()) }
        return null
    }

    /** The text to match apps against: "open camera" looks for "camera". */
    fun appQuery(text: String): String = OPEN.matchEntire(text.trim())?.groupValues?.get(1)?.trim() ?: text

    /** "5 min", "1h30", "90s", "1 hour 20 minutes", "10" (minutes). Null when it is not a duration. */
    fun duration(text: String): Int? {
        val s = text.trim().lowercase()
        if (s.isEmpty() || s.length > 40) return null
        var total = 0.0
        var pos = 0
        var any = false
        for (m in DURATION_PART.findAll(s)) {
            // Only spaces and "and"/"e" between the parts.
            val gap = s.substring(pos, m.range.first).trim()
            if (gap.isNotEmpty() && gap != "and" && gap != "e") return null
            pos = m.range.last + 1
            val n = m.groupValues[1].replace(',', '.').toDoubleOrNull() ?: return null
            val unit = when (m.groupValues[2].firstOrNull()) {
                'h' -> 'h'
                's' -> 's'
                'm' -> 'm'
                // No unit: minutes (also the minutes after hours, "1h30").
                else -> 'm'
            }
            total += n * when (unit) {
                'h' -> 3600
                's' -> 1
                else -> 60
            }
            any = true
        }
        if (!any || s.substring(pos).isNotBlank()) return null
        val secs = total.toInt()
        return secs.takeIf { it in 1..(24 * 3600) }
    }

    private fun alarm(m: MatchResult): Command.Alarm? {
        var h = m.groupValues[1].toIntOrNull() ?: return null
        val min = m.groupValues[2].takeIf { it.isNotEmpty() }?.toIntOrNull() ?: 0
        when (m.groupValues[3].lowercase()) {
            "pm" -> if (h < 12) h += 12
            "am" -> if (h == 12) h = 0
        }
        if (h !in 0..23 || min !in 0..59) return null
        return Command.Alarm(h, min)
    }

    /** Words people use for things an app's shortcuts name differently ("stories" is Instagram's "Camera"). */
    val SYNONYMS: Map<String, List<String>> = mapOf(
        "story" to listOf("story", "stories", "camera", "status"),
        "stories" to listOf("story", "stories", "camera", "status"),
        "status" to listOf("status", "story", "camera"),
        "dm" to listOf("direct", "messages", "inbox", "chats", "message"),
        "dms" to listOf("direct", "messages", "inbox", "chats", "message"),
        "direct" to listOf("direct", "messages", "inbox"),
        "messages" to listOf("messages", "direct", "inbox", "chats"),
        "chat" to listOf("chat", "chats", "messages", "direct"),
        "chats" to listOf("chats", "messages", "direct"),
        "inbox" to listOf("inbox", "messages", "direct"),
        "post" to listOf("post", "new post", "create", "upload"),
        "upload" to listOf("upload", "new post", "create"),
        "reels" to listOf("reels", "reel", "shorts", "videos"),
        "shorts" to listOf("shorts", "reels"),
        "scan" to listOf("scan", "qr", "camera"),
        "qr" to listOf("qr", "scan"),
        "camera" to listOf("camera", "story"),
        "search" to listOf("search", "find"),
        "notifications" to listOf("activity", "notifications"),
        "activity" to listOf("activity", "notifications"),
        "pix" to listOf("pix", "pay", "transfer"),
        "pay" to listOf("pay", "pix", "payment"),
        // Portuguese.
        "storys" to listOf("story", "stories", "camera", "status"),
        "mensagens" to listOf("messages", "direct", "inbox", "chats", "mensagens", "conversas"),
        "conversas" to listOf("chats", "conversas", "messages"),
        "postar" to listOf("new post", "post", "create", "publicar"),
        "publicar" to listOf("new post", "post", "create", "publicar"),
        "escanear" to listOf("scan", "qr", "camera", "escanear"),
        "notificacoes" to listOf("activity", "notifications", "notificacoes", "atividade"),
        "pagar" to listOf("pay", "pix", "pagar", "pagamento"),
        "transferir" to listOf("transfer", "pix", "transferir"),
    )
}
