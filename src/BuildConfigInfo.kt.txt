package io.github.jacomartins.canyonui

import android.content.Context
import android.os.Build
import android.content.pm.PackageManager

/** Version info read from the installed package (avoids enabling the BuildConfig feature). */
object BuildConfigInfo {
    fun versionName(context: Context): String = try {
        val pm = context.packageManager
        val info = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            pm.getPackageInfo(context.packageName, PackageManager.PackageInfoFlags.of(0))
        } else {
            @Suppress("DEPRECATION")
            pm.getPackageInfo(context.packageName, 0)
        }
        info.versionName ?: "?"
    } catch (_: PackageManager.NameNotFoundException) {
        "?"
    }
}
