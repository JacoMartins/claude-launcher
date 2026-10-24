package io.github.jacomartins.canyonui.model

/**
 * Sorts apps into home screen categories, offline. A normal launcher has no app store data, so
 * the sources are, in order:
 *
 * 1. A small table of well-known apps (exact package names).
 * 2. The category the app declares itself (`android:appCategory`, Android 8+), passed in as
 *    [declared] using the `ApplicationInfo.CATEGORY_*` values.
 * 3. Package name words (bank, pay, game, ...).
 *
 * Apps that none of these place return null: they stay out of the category folders.
 */
object AppCategories {

    enum class Category(val key: String) {
        COMMUNICATION("communication"),
        ENTERTAINMENT("entertainment"),
        GAMES("games"),
        CREATION("creation"),
        PRODUCTIVITY("productivity"),
        EDUCATION_AI("education_ai"),
        TRIP("trip"),
        WELLNESS("wellness"),
        FINANCE("finance"),
        SHOPPING("shopping"),
        NEWS("news"),
        TOOLS("tools"),

        /** Where apps without a category go when organizing; never returned by [categorize]. */
        OTHERS("others"),
        ;

        companion object {
            fun fromKey(key: String?) = entries.firstOrNull { it.key == key }
        }
    }

    // ApplicationInfo.CATEGORY_* values (stable platform constants).
    private const val GAME = 0
    private const val AUDIO = 1
    private const val VIDEO = 2
    private const val IMAGE = 3
    private const val SOCIAL = 4
    private const val NEWS_DECL = 5
    private const val MAPS = 6
    private const val PRODUCTIVITY_DECL = 7
    private const val ACCESSIBILITY = 8

    fun categorize(packageName: String, declared: Int): Category? {
        KNOWN[packageName]?.let { return it }
        // Families of packages (Samsung system apps, Google apps) by prefix.
        PREFIXES.firstOrNull { (prefix, _) -> packageName.startsWith(prefix) }?.let { return it.second }
        when (declared) {
            GAME -> return Category.GAMES
            AUDIO, VIDEO -> return Category.ENTERTAINMENT
            IMAGE -> return Category.CREATION
            SOCIAL -> return Category.COMMUNICATION
            NEWS_DECL -> return Category.NEWS
            MAPS -> return Category.TRIP
            PRODUCTIVITY_DECL -> return Category.PRODUCTIVITY
            ACCESSIBILITY -> return Category.TOOLS
        }
        val p = packageName.lowercase()
        return WORDS.firstOrNull { (words, _) -> words.any { it in p } }?.second
    }

    // Whole words or clear stems only: short fragments ("tv", "med", "bus") match unrelated names.
    private val WORDS: List<Pair<List<String>, Category>> = listOf(
        listOf("bank", "banco", "wallet", ".pay", "pay.", "finance", "invest", "crypto") to Category.FINANCE,
        listOf(".game", "games", "game.") to Category.GAMES,
        listOf("shopping", ".shop", "shop.") to Category.SHOPPING,
        listOf("health", "fitness", "workout", "saude", "meditat") to Category.WELLNESS,
        listOf("music", "podcast", "radio") to Category.ENTERTAINMENT,
        listOf("messenger", "messag", ".chat", "chat.") to Category.COMMUNICATION,
        listOf("photo", "camera") to Category.CREATION,
        listOf("learn", "school", "course", "study") to Category.EDUCATION_AI,
        listOf("taxi", "transit", "travel", "flight") to Category.TRIP,
        listOf(".news", "news.") to Category.NEWS,
    )

    private val PREFIXES: List<Pair<String, Category>> = listOf(
        "com.google.android.apps.docs" to Category.PRODUCTIVITY,
        "com.google.android.apps.tasks" to Category.PRODUCTIVITY,
        "com.microsoft.office" to Category.PRODUCTIVITY,
        "com.adobe." to Category.CREATION,
    )

    private val KNOWN: Map<String, Category> = buildMap {
        fun put(c: Category, vararg pkgs: String) = pkgs.forEach { put(it, c) }
        put(
            Category.COMMUNICATION,
            "com.whatsapp", "com.whatsapp.w4b", "org.telegram.messenger", "org.thoughtcrime.securesms",
            "com.discord", "com.facebook.katana", "com.facebook.orca", "com.instagram.android",
            "com.instagram.barcelona", "com.linkedin.android", "com.pinterest", "com.reddit.frontpage",
            "com.zhiliaoapp.musically", "com.ss.android.ugc.trill", "com.twitter.android", "com.snapchat.android",
            "com.bluesky.app", "xyz.blueskyweb.app", "com.google.android.gm", "com.samsung.android.messaging",
            "com.google.android.apps.messaging", "com.samsung.android.dialer", "com.google.android.dialer",
            "com.samsung.android.app.contacts", "com.google.android.contacts", "us.zoom.videomeetings",
            "com.microsoft.teams", "com.skype.raider", "com.google.android.apps.tachyon",
        )
        put(
            Category.ENTERTAINMENT,
            "com.spotify.music", "com.google.android.youtube", "com.google.android.apps.youtube.music",
            "com.netflix.mediaclient", "com.amazon.avod.thirdpartyclient", "com.disney.disneyplus",
            "com.hbo.hbonow", "tv.twitch.android.app", "deezer.android.app", "com.shazam.android",
            "com.soundcloud.android", "tv.pluto.android", "com.globo.globotv",
        )
        put(
            Category.CREATION,
            "com.canva.editor", "com.lemon.lvoverseas", "com.picsart.studio", "com.niksoftware.snapseed",
            "com.imageline.FLM", "com.github.android", "com.sec.android.app.camera", "com.google.android.GoogleCamera",
            "com.sec.android.gallery3d", "com.google.android.apps.photos", "com.vsco.cam", "com.instagram.layout",
            "com.adobe.lrmobile", "com.adobe.spark.post",
        )
        put(
            Category.PRODUCTIVITY,
            "com.google.android.calendar", "com.samsung.android.calendar", "com.google.android.apps.docs",
            "notion.id", "com.todoist", "com.ticktick.task", "com.microsoft.todos", "com.evernote",
            "com.google.android.keep", "com.samsung.android.app.reminder", "com.dropbox.android",
            "com.microsoft.skydrive", "com.trello", "com.Slack", "com.google.android.googlequicksearchbox",
            "com.sec.android.app.sbrowser", "com.android.chrome", "org.mozilla.firefox", "com.brave.browser",
        )
        put(
            Category.EDUCATION_AI,
            "com.openai.chatgpt", "com.google.android.apps.bard", "com.anthropic.claude", "com.deepseek.chat",
            "ai.perplexity.app.android", "com.microsoft.copilot", "com.google.android.apps.classroom",
            "com.duolingo", "com.moodle.moodlemobile", "org.khanacademy.android", "com.quizlet.quizletandroid",
            "com.google.android.apps.labs.language.tailwind",
        )
        put(
            Category.TRIP,
            "com.ubercab", "com.taxis99", "com.waze", "com.google.android.apps.maps", "com.moovit.android",
            "com.citymapper.app.release", "com.airbnb.android", "com.booking",
            "com.google.android.apps.travel.onthego",
        )
        put(
            Category.WELLNESS,
            "com.sec.android.app.shealth", "com.strava", "com.google.android.apps.fitness", "com.calm.android",
            "com.getsomeheadspace.android", "com.myfitnesspal.android", "com.samsung.android.forest",
        )
        put(
            Category.FINANCE,
            "com.nu.production", "com.itau", "br.com.bb.android", "com.bradesco", "com.santander.app",
            "com.picpay", "com.mercadopago.wallet", "com.paypal.android.p2pmobile", "br.com.intermedium",
            "com.c6bank.app", "com.google.android.apps.walletnfcrel", "com.samsung.android.spay",
        )
        put(
            Category.SHOPPING,
            "com.amazon.mShop.android.shopping", "com.alibaba.aliexpresshd", "com.shopee.br", "com.mercadolibre",
            "com.zzkko", "com.einnovation.temu", "com.samsung.shop",
        )
        // Widely used apps, added so most phones need few manual choices.
        put(
            Category.COMMUNICATION,
            "com.google.android.apps.meetings", "com.google.android.apps.dynamite", "com.viber.voip", "jp.naver.line.android",
            "com.kakao.talk", "com.tencent.mm", "com.microsoft.office.outlook", "com.samsung.android.email.provider",
            "com.tumblr", "com.quora.android", "com.facebook.lite", "com.instagram.lite", "com.facebook.mlite", "com.truecaller",
            "com.vkontakte.android",
        )
        put(
            Category.ENTERTAINMENT,
            "com.amazon.mp3", "com.apple.android.music", "com.google.android.videos", "com.crunchyroll.crunchyroid",
            "com.hulu.plus", "com.wbd.stream", "com.google.android.apps.youtube.kids", "com.audible.application",
            "com.pandora.android", "com.aspiro.tidal", "com.samsung.android.tvplus", "com.mxtech.videoplayer.ad",
            "org.videolan.vlc",
        )
        put(
            Category.GAMES,
            "com.supercell.clashofclans", "com.supercell.clashroyale", "com.supercell.brawlstars", "com.king.candycrushsaga",
            "com.mojang.minecraftpe", "com.roblox.client", "com.dts.freefireth", "com.dts.freefiremax", "com.tencent.ig",
            "com.activision.callofduty.shooter", "com.kiloo.subwaysurf", "com.miniclip.eightballpool",
            "com.innersloth.spacemafia", "com.chess", "org.lichess.mobileapp", "com.nianticlabs.pokemongo",
        )
        put(
            Category.CREATION,
            "com.camerasideas.instashot", "com.nexstreaming.app.kinemasterfree", "com.alightcreative.motion", "com.figma.mirror",
        )
        put(
            Category.PRODUCTIVITY,
            "com.samsung.android.app.notes", "com.adobe.reader", "md.obsidian", "com.anydo", "com.asana.app",
        )
        put(
            Category.EDUCATION_AI,
            "com.microsoft.bing", "ai.x.grok", "com.microblink.photomath", "co.brainly", "org.coursera.android",
            "com.udemy.android", "com.babbel.mobile.android.en", "com.memrise.android.memrisecompanion",
        )
        put(
            Category.TRIP,
            "me.lyft.android", "ee.mtakso.client", "com.expedia.bookings", "com.tripadvisor.tripadvisor", "com.google.earth",
        )
        put(
            Category.WELLNESS,
            "com.fitbit.FitbitMobile", "com.google.android.apps.healthdata", "com.nike.plusgps", "com.runtastic.android",
            "org.iggymedia.periodtracker", "com.google.android.apps.wellbeing",
        )
        put(
            Category.FINANCE,
            "com.binance.dev", "com.coinbase.android", "com.revolut.revolut", "com.transferwise.android", "com.venmo",
            "com.squareup.cash", "com.google.android.apps.nbu.paisa.user",
        )
        put(
            Category.SHOPPING,
            "com.ebay.mobile", "com.walmart.android", "com.etsy.android", "com.luizalabs.mlapp", "com.contextlogic.wish",
            "com.ubercab.eats", "br.com.brainweb.ifood", "com.grability.rappi", "com.dd.doordash", "com.instacart.client",
        )
        put(
            Category.NEWS,
            "com.nytimes.android", "bbc.mobile.news.ww", "com.cnn.mobile.android.phone", "com.microsoft.amp.apps.bingnews",
            "wp.wattpad", "com.google.android.apps.books",
        )
        put(
            Category.TOOLS,
            "com.android.vending", "com.sec.android.app.samsungapps", "com.google.android.apps.authenticator2",
            "com.x8bit.bitwarden", "com.google.android.apps.translate", "com.google.android.apps.chromecast.app",
            "com.samsung.android.app.watchmanager", "com.google.android.apps.recorder", "org.zwanoo.android.speedtest",
            "com.google.android.apps.subscriptions.red",
        )
        put(Category.NEWS, "com.google.android.apps.magazines", "flipboard.app", "com.medium.reader", "com.amazon.kindle")
        put(
            Category.TOOLS,
            "com.android.settings", "com.sec.android.app.popupcalculator", "com.google.android.calculator",
            "com.sec.android.app.clockpackage", "com.google.android.deskclock", "com.sec.android.app.myfiles",
            "com.google.android.apps.nbu.files", "com.samsung.android.lool", "com.sec.android.app.voicenote",
            "com.samsung.android.app.smartcapture", "com.samsung.android.oneconnect", "com.sec.android.easyMover",
            "com.samsung.android.app.galaxyfinder", "com.samsung.android.voc", "com.samsung.android.app.tips",
            "com.sec.android.daemonapp", "com.samsung.android.app.find",
        )
    }
}
