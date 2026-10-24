package io.github.jacomartins.canyonui.data

import android.content.Context
import android.content.Intent
import android.net.Uri
import android.util.Log
import io.github.jacomartins.canyonui.HomeActivity
import io.github.jacomartins.canyonui.LauncherApp
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.io.File
import java.util.zip.ZipEntry
import java.util.zip.ZipInputStream
import java.util.zip.ZipOutputStream

/**
 * Backs up and restores the whole launcher in one file (a zip the user saves where they like):
 * both layouts (pages, folders, widgets, stacks), all settings, and how often apps are opened.
 * Widgets come back on the same phone; on another phone each shows a card to set it up again.
 */
object Backup {
    private const val TAG = "Backup"
    private const val MARKER = "canyon-ui-backup"
    private val DATABASES = listOf("launcher.db", "launcher_home_only.db")

    /** The files kept, by their name inside the backup. */
    private fun files(context: Context): Map<String, File> = buildMap {
        for (db in DATABASES) put("databases/$db", context.getDatabasePath(db))
        put("datastore/settings.preferences_pb", File(context.filesDir, "datastore/settings.preferences_pb"))
        put("shared_prefs/app_usage.xml", File(context.dataDir, "shared_prefs/app_usage.xml"))
    }

    suspend fun export(app: LauncherApp, uri: Uri): Boolean = withContext(Dispatchers.IO) {
        try {
            // Everything written so far goes into the database files themselves.
            for (db in app.dbs.values) db.openHelper.writableDatabase.query("PRAGMA wal_checkpoint(FULL)").use { it.moveToFirst() }
            app.contentResolver.openOutputStream(uri)?.use { out ->
                ZipOutputStream(out).use { zip ->
                    zip.putNextEntry(ZipEntry(MARKER))
                    zip.write("1".toByteArray())
                    zip.closeEntry()
                    for ((name, file) in files(app)) {
                        if (!file.exists()) continue
                        zip.putNextEntry(ZipEntry(name))
                        file.inputStream().use { it.copyTo(zip) }
                        zip.closeEntry()
                    }
                }
            } ?: return@withContext false
            true
        } catch (e: Exception) {
            Log.w(TAG, "Backup failed", e)
            false
        }
    }

    /**
     * Restores a backup made by [export], then restarts the launcher so it reads it. Returns
     * false (changing nothing) when the file is not a Canyon UI backup.
     */
    suspend fun restore(app: LauncherApp, uri: Uri): Boolean = withContext(Dispatchers.IO) {
        try {
            val entries = HashMap<String, ByteArray>()
            app.contentResolver.openInputStream(uri)?.use { input ->
                ZipInputStream(input).use { zip ->
                    while (true) {
                        val e = zip.nextEntry ?: break
                        if (!e.isDirectory) entries[e.name] = zip.readBytes()
                    }
                }
            } ?: return@withContext false
            if (MARKER !in entries) return@withContext false
            val targets = files(app)
            for (db in app.dbs.values) db.close()
            for ((name, file) in targets) {
                val data = entries[name] ?: continue
                file.parentFile?.mkdirs()
                if (name.startsWith("databases/")) {
                    File(file.path + "-wal").delete()
                    File(file.path + "-shm").delete()
                }
                file.writeBytes(data)
            }
            restart(app)
            true
        } catch (e: Exception) {
            Log.w(TAG, "Restore failed", e)
            false
        }
    }

    /** Starts home again in a fresh process (the restored files are read from the start). */
    private fun restart(context: Context) {
        context.startActivity(
            Intent(context, HomeActivity::class.java).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK),
        )
        Runtime.getRuntime().exit(0)
    }
}
