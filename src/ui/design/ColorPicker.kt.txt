package io.github.jacomartins.canyonui.ui.design

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.background
import androidx.compose.foundation.gestures.detectHorizontalDragGestures
import androidx.compose.foundation.gestures.detectTapGestures
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.BoxWithConstraints
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.offset
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableFloatStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.Brush
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.toArgb
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.progressBarRangeInfo
import androidx.compose.ui.semantics.ProgressBarRangeInfo
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.semantics.setProgress
import androidx.compose.ui.unit.dp

/**
 * A colour picker with transparency: hue, saturation, brightness and opacity sliders over a
 * preview. [onCommit] runs when a slider is let go (not on every frame of a drag) with the
 * colour as ARGB; [onChange], when given, on every change. [labels] name the four sliders for
 * accessibility. With [initial] 0 it starts on [startColor]'s colour.
 */
@Composable
fun ColorPicker(
    initial: Int,
    labels: List<String>,
    onCommit: (Int) -> Unit,
    modifier: Modifier = Modifier,
    onChange: ((Int) -> Unit)? = null,
) {
    val start = remember(initial) {
        FloatArray(3).also { android.graphics.Color.colorToHSV(if (initial == 0) DEFAULT else initial, it) }
    }
    var hue by remember(initial) { mutableFloatStateOf(start[0] / 360f) }
    var sat by remember(initial) { mutableFloatStateOf(start[1]) }
    var value by remember(initial) { mutableFloatStateOf(start[2]) }
    var alpha by remember(initial) { mutableFloatStateOf(if (initial == 0) DEFAULT_ALPHA else ((initial ushr 24) / 255f)) }

    // The full range: an opaque colour stays opaque on the folder.
    fun argb(): Int = android.graphics.Color.HSVToColor((alpha * 255).toInt(), floatArrayOf(hue * 360f, sat, value))
    fun commit() = onCommit(argb())

    val full = Color(android.graphics.Color.HSVToColor(floatArrayOf(hue * 360f, 1f, 1f)))
    val opaque = Color(android.graphics.Color.HSVToColor(floatArrayOf(hue * 360f, sat, value)))

    Column(modifier, verticalArrangement = Arrangement.spacedBy(Spacing.sm)) {
        Row(verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.spacedBy(Spacing.smMd)) {
            Box(Modifier.size(36.dp).clip(CircleShape)) {
                Checker(Modifier.fillMaxSize())
                Box(Modifier.fillMaxSize().background(Color(argb())))
            }
            Text(
                "#%08X".format(argb()),
                style = MaterialTheme.typography.labelLarge,
                color = MaterialTheme.colorScheme.onSurface,
            )
        }
        GradientSlider(hue, labels[0], Brush.horizontalGradient(HUES), { hue = it; onChange?.invoke(argb()) }, ::commit)
        GradientSlider(sat, labels[1], Brush.horizontalGradient(listOf(Color.White, full)), { sat = it; onChange?.invoke(argb()) }, ::commit)
        GradientSlider(value, labels[2], Brush.horizontalGradient(listOf(Color.Black, full)), { value = it; onChange?.invoke(argb()) }, ::commit)
        GradientSlider(alpha, labels[3], Brush.horizontalGradient(listOf(opaque.copy(alpha = 0f), opaque)), { alpha = it; onChange?.invoke(argb()) }, ::commit, checker = true)
    }
}

/** A pill track filled with [brush] and a round thumb; drag or tap to set [value] in 0..1. */
@Composable
private fun GradientSlider(
    value: Float,
    label: String,
    brush: Brush,
    onChange: (Float) -> Unit,
    onDone: () -> Unit,
    checker: Boolean = false,
) {
    val thumb = 24.dp
    BoxWithConstraints(
        Modifier
            .fillMaxWidth()
            .height(28.dp)
            .semantics {
                contentDescription = label
                progressBarRangeInfo = ProgressBarRangeInfo(value, 0f..1f)
                setProgress { v -> onChange(v.coerceIn(0f, 1f)); onDone(); true }
            },
    ) {
        val widthPx = constraints.maxWidth.toFloat()
        val thumbPx = with(LocalDensity.current) { thumb.toPx() }
        fun at(x: Float) = ((x - thumbPx / 2f) / (widthPx - thumbPx)).coerceIn(0f, 1f)
        Box(
            Modifier
                .fillMaxSize()
                .padding(vertical = 4.dp)
                .clip(CircleShape)
                .oneDepthBorder()
                .pointerInput(widthPx) {
                    detectTapGestures { onChange(at(it.x)); onDone() }
                }
                .pointerInput(widthPx) {
                    detectHorizontalDragGestures(
                        onDragEnd = onDone,
                        onDragCancel = onDone,
                    ) { change, _ -> onChange(at(change.position.x)) }
                },
        ) {
            if (checker) Checker(Modifier.fillMaxSize())
            Box(Modifier.fillMaxSize().background(brush))
        }
        val x = with(LocalDensity.current) { ((widthPx - thumbPx) * value).toDp() }
        Box(
            Modifier
                .offset(x = x)
                .size(thumb)
                .align(Alignment.CenterStart)
                .clip(CircleShape)
                .background(Color.White)
                .oneDepthBorder(),
        )
    }
}

/** Grey checkerboard behind transparent colours. */
@Composable
private fun Checker(modifier: Modifier) {
    Canvas(modifier) {
        val s = 6.dp.toPx()
        drawRect(Color(0xFFE0E0E0))
        var y = 0f
        var row = 0
        while (y < size.height) {
            var x = if (row % 2 == 0) 0f else s
            while (x < size.width) {
                drawRect(Color(0xFFBDBDBD), Offset(x, y), Size(s, s))
                x += 2 * s
            }
            y += s
            row++
        }
    }
}

private val HUES = listOf(
    Color(0xFFFF0000), Color(0xFFFFFF00), Color(0xFF00FF00), Color(0xFF00FFFF),
    Color(0xFF0000FF), Color(0xFFFF00FF), Color(0xFFFF0000),
)
private val DEFAULT = Color(0xFF5B8DEF).toArgb()

/** The colour [ColorPicker] shows for [initial]: [initial] itself, or its starting colour for 0. */
fun startColor(initial: Int): Int = if (initial != 0) initial else {
    val hsv = FloatArray(3).also { android.graphics.Color.colorToHSV(DEFAULT, it) }
    android.graphics.Color.HSVToColor((DEFAULT_ALPHA * 255).toInt(), hsv)
}
private const val DEFAULT_ALPHA = 0.65f
