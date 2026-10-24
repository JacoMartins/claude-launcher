package io.github.jacomartins.canyonui.data

import androidx.annotation.StringRes
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.model.AppCategories.Category

/** A category's name, as a string resource (folder names, the category chooser). */
@StringRes
fun Category.titleRes(): Int = when (this) {
    Category.COMMUNICATION -> R.string.category_communication
    Category.ENTERTAINMENT -> R.string.category_entertainment
    Category.GAMES -> R.string.category_games
    Category.CREATION -> R.string.category_creation
    Category.PRODUCTIVITY -> R.string.category_productivity
    Category.EDUCATION_AI -> R.string.category_education_ai
    Category.TRIP -> R.string.category_trip
    Category.WELLNESS -> R.string.category_wellness
    Category.FINANCE -> R.string.category_finance
    Category.SHOPPING -> R.string.category_shopping
    Category.NEWS -> R.string.category_news
    Category.TOOLS -> R.string.category_tools
    Category.OTHERS -> R.string.category_others
}
