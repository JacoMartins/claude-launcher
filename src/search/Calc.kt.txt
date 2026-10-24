package io.github.jacomartins.canyonui.search

import java.math.BigDecimal
import java.math.MathContext
import kotlin.math.pow

/**
 * The search's calculator: + − × ÷ % ^, brackets and unary minus over decimal numbers. Pure
 * Kotlin. Only text that is clearly a calculation gets an answer (a number on its own does not).
 */
object Calc {

    /** The value of [text], or null when it is not a calculation or cannot be worked out. */
    fun evaluate(text: String): Double? {
        val src = normalize(text) ?: return null
        val p = Parser(src)
        val v = try {
            p.expression().also { p.skipSpaces() }
        } catch (e: IllegalArgumentException) {
            return null
        }
        if (!p.atEnd() || !p.sawOperator) return null
        return v.takeIf { it.isFinite() }
    }

    /** [value] for display: up to 10 significant digits, no trailing zeros, no exponent for usual sizes. */
    fun format(value: Double): String {
        if (value == 0.0) return "0"
        val bd = BigDecimal(value).round(MathContext(10)).stripTrailingZeros()
        val abs = bd.abs()
        return if (abs >= BigDecimal("1e15") || abs < BigDecimal("1e-6")) bd.toString() else bd.toPlainString()
    }

    private fun normalize(text: String): String? {
        var s = text.trim()
        if (s.isEmpty() || s.length > 120) return null
        s = s.replace('×', '*').replace('x', '*').replace('X', '*').replace('÷', '/').replace('−', '-').replace(':', '/')
        if (s.endsWith("=")) s = s.dropLast(1)
        // A comma is the decimal mark when there is no point (e.g. "2,5 * 4").
        if (!s.contains('.')) s = s.replace(',', '.')
        if (s.any { !(it.isDigit() || it in "+-*/%^().  ") }) return null
        if (s.none { it.isDigit() }) return null
        return s
    }

    private class Parser(private val s: String) {
        private var i = 0
        var sawOperator = false
            private set

        fun atEnd() = i >= s.length

        fun skipSpaces() {
            while (i < s.length && s[i] == ' ') i++
        }

        private fun peek(): Char? {
            skipSpaces()
            return s.getOrNull(i)
        }

        // expression := term (('+' | '-') term)*
        fun expression(): Double {
            var v = term()
            while (true) {
                when (peek()) {
                    '+' -> { i++; sawOperator = true; v += term() }
                    '-' -> { i++; sawOperator = true; v -= term() }
                    else -> return v
                }
            }
        }

        // term := power (('*' | '/') power | '%')*
        private fun term(): Double {
            var v = power()
            while (true) {
                when (peek()) {
                    '*' -> { i++; sawOperator = true; v *= power() }
                    '/' -> {
                        i++; sawOperator = true
                        val d = power()
                        require(d != 0.0)
                        v /= d
                    }
                    '%' -> { i++; sawOperator = true; v /= 100.0 }
                    else -> return v
                }
            }
        }

        // power := unary ('^' power)?   (right-associative)
        private fun power(): Double {
            val base = unary()
            if (peek() == '^') {
                i++
                sawOperator = true
                return base.pow(power())
            }
            return base
        }

        private fun unary(): Double = when (peek()) {
            '-' -> { i++; -unary() }
            '+' -> { i++; unary() }
            else -> primary()
        }

        private fun primary(): Double {
            if (peek() == '(') {
                i++
                val v = expression()
                require(peek() == ')')
                i++
                // Brackets count as a calculation: "(2)" alone is not, "(2)+1" is.
                return v
            }
            skipSpaces()
            val start = i
            while (i < s.length && (s[i].isDigit() || s[i] == '.')) i++
            require(i > start)
            return s.substring(start, i).toDoubleOrNull() ?: throw IllegalArgumentException()
        }
    }
}
