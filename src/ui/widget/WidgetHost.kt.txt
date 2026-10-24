package io.github.jacomartins.canyonui.ui.widget

import android.annotation.SuppressLint
import android.appwidget.AppWidgetHost
import android.appwidget.AppWidgetHostView
import android.appwidget.AppWidgetProviderInfo
import android.content.Context
import android.graphics.Canvas
import android.graphics.drawable.GradientDrawable
import android.util.Log
import android.view.Gravity
import android.view.MotionEvent
import android.view.View
import android.view.ViewConfiguration
import android.widget.RemoteViews
import android.widget.TextView
import io.github.jacomartins.canyonui.R
import kotlin.math.abs

/**
 * The launcher's widget host. Every call into the system service is guarded: a misbehaving
 * provider or a large update (TransactionTooLargeException) must never take the home screen down.
 */
class LauncherWidgetHost(context: Context, private val onProvidersChanged: () -> Unit) :
    AppWidgetHost(context.applicationContext, HOST_ID) {

    override fun onCreateView(context: Context, appWidgetId: Int, appWidget: AppWidgetProviderInfo?): AppWidgetHostView =
        LauncherWidgetHostView(context)

    override fun onProvidersChanged() = onProvidersChanged.invoke()

    fun startListeningSafely() {
        try {
            startListening()
        } catch (e: RuntimeException) {
            Log.w(TAG, "startListening failed", e)
        }
    }

    fun stopListeningSafely() {
        try {
            stopListening()
        } catch (e: RuntimeException) {
            Log.w(TAG, "stopListening failed", e)
        }
    }

    /** A host view for [appWidgetId], or null if the provider cannot be shown at all. */
    fun createViewSafely(context: Context, appWidgetId: Int, info: AppWidgetProviderInfo): LauncherWidgetHostView? = try {
        createView(context, appWidgetId, info) as LauncherWidgetHostView
    } catch (e: RuntimeException) {
        Log.w(TAG, "createView failed for ${info.provider}", e)
        null
    }

    fun deleteIdSafely(appWidgetId: Int) {
        try {
            deleteAppWidgetId(appWidgetId)
        } catch (e: RuntimeException) {
            Log.w(TAG, "deleteAppWidgetId failed", e)
        }
    }

    companion object {
        const val HOST_ID = 1024
        private const val TAG = "WidgetHost"
    }
}

/**
 * A widget on the home screen.
 *
 * - Long-press is detected here, before the widget's own views (lists, buttons) can claim the
 *   touch, and then the rest of the gesture is taken from them so it can become a drag.
 * - In edit mode ([blockContent]) the widget's content gets no touches; a tap calls the click
 *   listener instead (which leaves edit mode).
 * - Crash protection: an exception while measuring, laying out or drawing the provider's views
 *   replaces them with an error card instead of crashing the launcher.
 */
@SuppressLint("ViewConstructor")
class LauncherWidgetHostView(context: Context) : AppWidgetHostView(context) {

    var blockContent = false

    private val slop = ViewConfiguration.get(context).scaledTouchSlop
    private val longPressTimeout = ViewConfiguration.getLongPressTimeout().toLong()
    private var downX = 0f
    private var downY = 0f
    private var longPressed = false
    private var moved = false
    private var crashed = false

    private val longPress = Runnable {
        if (isAttachedToWindow && isPressedForLongPress) {
            longPressed = true
            performLongClick()
        }
    }
    private var isPressedForLongPress = false

    init {
        // The widget's corner radius and background come from the provider (Android 12+).
        clipChildren = true
        io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.track(this)
    }

    private val cardPath = android.graphics.Path()

    override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                downX = ev.x; downY = ev.y
                longPressed = false
                moved = false
                isPressedForLongPress = true
                postDelayed(longPress, longPressTimeout)
            }
            MotionEvent.ACTION_MOVE -> if (!moved && (abs(ev.x - downX) > slop || abs(ev.y - downY) > slop)) {
                moved = true
                cancelLongPress()
            }
            MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> cancelLongPress()
        }
        return super.dispatchTouchEvent(ev)
    }

    override fun cancelLongPress() {
        isPressedForLongPress = false
        removeCallbacks(longPress)
        super.cancelLongPress()
    }

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean = longPressed || blockContent

    @SuppressLint("ClickableViewAccessibility") // performClick is called; long-press is also a menu action.
    override fun onTouchEvent(ev: MotionEvent): Boolean {
        if (ev.actionMasked == MotionEvent.ACTION_UP && blockContent && !longPressed && !moved) performClick()
        // Claim the gesture so it is not handed to the page underneath (no empty-space long-press).
        return true
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        try {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec)
        } catch (e: RuntimeException) {
            setMeasuredDimension(MeasureSpec.getSize(widthMeasureSpec), MeasureSpec.getSize(heightMeasureSpec))
            onCrash(e)
        }
    }

    override fun onLayout(changed: Boolean, left: Int, top: Int, right: Int, bottom: Int) {
        try {
            super.onLayout(changed, left, top, right, bottom)
        } catch (e: RuntimeException) {
            onCrash(e)
        }
        // The widget's visible card may have moved or changed size.
        WidgetFrame.refresh(this)
    }

    override fun dispatchDraw(canvas: Canvas) {
        // Frosted and glass: the blurred wallpaper behind the widget's own card, so a see-through
        // widget background frosts like the launcher's surfaces.
        val cardRadius = WidgetFrame.cardPath(this, cardPath)
        if (cardRadius >= 0f) {
            io.github.jacomartins.canyonui.ui.design.WallpaperBackdrop.draw(canvas, this, cardPath, cardRadius)
        }
        try {
            super.dispatchDraw(canvas)
        } catch (e: RuntimeException) {
            onCrash(e)
        }
    }

    override fun updateAppWidget(remoteViews: RemoteViews?) {
        crashed = false
        try {
            super.updateAppWidget(remoteViews)
        } catch (e: RuntimeException) {
            onCrash(e)
        }
    }

    private fun onCrash(e: RuntimeException) {
        Log.w("WidgetHostView", "Widget ${appWidgetInfo?.provider} failed; showing an error card", e)
        if (crashed) return
        crashed = true
        // Not during this layout or draw pass.
        post {
            removeAllViews()
            addView(errorView())
        }
    }

    override fun getErrorView(): View = errorView()

    private fun errorView(): View = TextView(context).apply {
        gravity = Gravity.CENTER
        setText(R.string.widget_error)
        textSize = 13f
        val d = resources.displayMetrics.density
        setPadding((12 * d).toInt(), (12 * d).toInt(), (12 * d).toInt(), (12 * d).toInt())
        setTextColor(0xFFFFFFFF.toInt())
        background = GradientDrawable().apply {
            cornerRadius = 20 * d
            setColor(0x66000000)
        }
    }
}
