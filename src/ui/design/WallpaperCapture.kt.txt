package io.github.jacomartins.canyonui.ui.design

import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.Service
import android.content.Context
import android.content.Intent
import android.content.pm.ServiceInfo
import android.graphics.Bitmap
import android.graphics.PixelFormat
import android.hardware.display.DisplayManager
import android.hardware.display.VirtualDisplay
import android.media.ImageReader
import android.media.projection.MediaProjection
import android.media.projection.MediaProjectionManager
import android.os.Build
import android.os.Handler
import android.os.IBinder
import android.os.Looper
import androidx.core.graphics.createBitmap
import io.github.jacomartins.canyonui.R

/**
 * Captures one frame of the screen with Android's screen capture (the user agrees in the system
 * dialog each time), while home shows only the wallpaper: the launcher's copy for the frosted and
 * glass effects when the wallpaper was set elsewhere or is live. Android requires a foreground
 * service for this; it runs for about a second and stops.
 */
class WallpaperCapture : Service() {

    private val handler = Handler(Looper.getMainLooper())
    private var projection: MediaProjection? = null
    private var display: VirtualDisplay? = null
    private var reader: ImageReader? = null
    private var done = false

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground(NOTIFICATION_ID, notification(), ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PROJECTION)
        val code = intent?.getIntExtra(EXTRA_CODE, 0) ?: 0
        @Suppress("DEPRECATION")
        val data = intent?.getParcelableExtra<Intent>(EXTRA_DATA)
        val mp = if (data != null) {
            try {
                getSystemService(MediaProjectionManager::class.java).getMediaProjection(code, data)
            } catch (e: Exception) {
                null
            }
        } else {
            null
        }
        if (mp == null) {
            finish(null)
            return START_NOT_STICKY
        }
        projection = mp
        // Android 14 asks for a callback before the capture starts.
        mp.registerCallback(object : MediaProjection.Callback() {
            override fun onStop() = finish(null)
        }, handler)
        // The consent dialog fades away, and home hides its icons and the system bars, first.
        handler.postDelayed(::capture, SETTLE_MS)
        handler.postDelayed({ finish(null) }, TIMEOUT_MS)
        return START_NOT_STICKY
    }

    private fun capture() {
        val mp = projection ?: return
        val (w, h) = WallpaperBackdrop.screenSize(this)
        val r = ImageReader.newInstance(w, h, PixelFormat.RGBA_8888, 2)
        reader = r
        var frames = 0
        r.setOnImageAvailableListener({ ir ->
            val image = ir.acquireLatestImage() ?: return@setOnImageAvailableListener
            image.use {
                // The first frames can still be the previous screen: take the third.
                if (++frames < FRAME) return@setOnImageAvailableListener
                val plane = it.planes[0]
                val rowPixels = plane.rowStride / plane.pixelStride
                val padded = createBitmap(rowPixels, h)
                padded.copyPixelsFromBuffer(plane.buffer)
                val shot = if (rowPixels == w) padded else Bitmap.createBitmap(padded, 0, 0, w, h)
                finish(shot)
            }
        }, handler)
        display = try {
            mp.createVirtualDisplay(
                "canyon-wallpaper", w, h, resources.displayMetrics.densityDpi,
                DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR, r.surface, null, handler,
            )
        } catch (e: Exception) {
            finish(null)
            null
        }
    }

    private fun finish(shot: Bitmap?) {
        if (done) return
        done = true
        handler.removeCallbacksAndMessages(null)
        display?.release()
        reader?.close()
        try {
            projection?.stop()
        } catch (e: Exception) {
            // Already stopped.
        }
        projection = null
        onResult?.invoke(shot)
        onResult = null
        stopForeground(STOP_FOREGROUND_REMOVE)
        stopSelf()
    }

    private fun notification(): Notification {
        val nm = getSystemService(NotificationManager::class.java)
        nm.createNotificationChannel(NotificationChannel(CHANNEL, getString(R.string.capture_channel), NotificationManager.IMPORTANCE_LOW))
        return Notification.Builder(this, CHANNEL)
            .setSmallIcon(R.drawable.ph_wallpaper)
            .setContentTitle(getString(R.string.capture_running))
            .build()
    }

    companion object {
        private const val CHANNEL = "wallpaper_capture"
        private const val NOTIFICATION_ID = 7
        private const val EXTRA_CODE = "code"
        private const val EXTRA_DATA = "data"
        private const val SETTLE_MS = 900L
        private const val TIMEOUT_MS = 6000L
        private const val FRAME = 3

        /** Receives the frame (null when it failed), on the main thread. */
        var onResult: ((Bitmap?) -> Unit)? = null

        /** The system's consent screen; ask for the whole screen where Android lets apps choose. */
        fun consentIntent(context: Context): Intent {
            val mpm = context.getSystemService(MediaProjectionManager::class.java)
            return if (Build.VERSION.SDK_INT >= 34) {
                mpm.createScreenCaptureIntent(android.media.projection.MediaProjectionConfig.createConfigForDefaultDisplay())
            } else {
                mpm.createScreenCaptureIntent()
            }
        }

        fun start(context: Context, resultCode: Int, data: Intent, result: (Bitmap?) -> Unit) {
            onResult = result
            context.startForegroundService(
                Intent(context, WallpaperCapture::class.java).putExtra(EXTRA_CODE, resultCode).putExtra(EXTRA_DATA, data),
            )
        }
    }
}
