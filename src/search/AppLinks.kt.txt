package io.github.jacomartins.canyonui.search

import android.content.Intent
import android.net.Uri
import androidx.core.net.toUri

/**
 * Public deep links of popular apps, so "uber <address>" opens Uber with that destination and
 * "waze <address>" starts the drive. Only links the apps document for other apps are used; each
 * intent names the app's package, and is offered only when that app is installed.
 */
object AppLinks {

    enum class Kind { RIDE, DRIVE, SEARCH }

    class Link(val pkg: String, val kind: Kind, private val build: (String) -> String) {
        fun intent(text: String): Intent = Intent(Intent.ACTION_VIEW, build(Uri.encode(text)).toUri()).setPackage(pkg)
    }

    val all: List<Link> = listOf(
        // Uber's universal link: pickup here, the destination as typed (Uber looks it up).
        Link("com.ubercab", Kind.RIDE) { "https://m.uber.com/ul/?action=setPickup&pickup=my_location&dropoff[formatted_address]=$it&dropoff[nickname]=$it" },
        // Waze's deep link: searches the place and starts the drive to the first result.
        Link("com.waze", Kind.DRIVE) { "https://waze.com/ul?q=$it&navigate=yes" },
        Link("com.google.android.apps.maps", Kind.DRIVE) { "google.navigation:q=$it" },
        Link("com.spotify.music", Kind.SEARCH) { "spotify:search:$it" },
        Link("com.google.android.youtube", Kind.SEARCH) { "https://www.youtube.com/results?search_query=$it" },
        Link("com.google.android.apps.youtube.music", Kind.SEARCH) { "https://music.youtube.com/search?q=$it" },
    )

    fun forApp(pkg: String): List<Link> = all.filter { it.pkg == pkg }

    /** Apps that take a destination, among [installed] packages: ride apps first, then navigation. */
    fun destinations(installed: Set<String>): List<Link> =
        all.filter { (it.kind == Kind.RIDE || it.kind == Kind.DRIVE) && it.pkg in installed }.sortedBy { it.kind.ordinal }
}
