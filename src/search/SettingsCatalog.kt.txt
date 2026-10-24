package io.github.jacomartins.canyonui.search

import android.content.Context
import android.content.Intent
import android.provider.Settings
import io.github.jacomartins.canyonui.R
import io.github.jacomartins.canyonui.SettingsActivity
import io.github.jacomartins.canyonui.WallpaperActivity

/**
 * A settings page search can open: the phone's (public `android.provider.Settings` actions, so
 * no maker's search is needed) or Canyon UI's own. [keywords] are other words people use for it,
 * in English and Portuguese (accent-free: matching ignores accents).
 */
class SettingPage(
    val label: Int,
    val icon: Int,
    val keywords: String,
    /** The phone's settings action, or null for one of Canyon UI's own screens. */
    val action: String?,
    val own: Class<*>? = null,
) {
    /** Its [ActionStats] key. */
    val key: String get() = "setting:" + (action ?: own?.simpleName)

    fun intent(context: Context): Intent =
        if (own != null) Intent(context, own) else Intent(action)
}

/** Every settings page search offers. Labels are short, as in the phone's settings. */
object SettingsCatalog {

    val pages: List<SettingPage> = listOf(
        SettingPage(R.string.set_wifi, R.drawable.ph_wifi_high, "wifi wi-fi wireless internet network hotspot internet rede roteador", Settings.ACTION_WIFI_SETTINGS),
        SettingPage(R.string.set_bluetooth, R.drawable.ph_bluetooth, "bluetooth pair headphones earbuds watch parear fone relogio", Settings.ACTION_BLUETOOTH_SETTINGS),
        SettingPage(R.string.set_network, R.drawable.ph_cell_signal_full, "mobile data sim network connections dados moveis chip rede conexoes", Settings.ACTION_WIRELESS_SETTINGS),
        SettingPage(R.string.set_data_usage, R.drawable.ph_cell_signal_full, "data usage mobile data saver uso de dados economia de dados", Settings.ACTION_DATA_USAGE_SETTINGS),
        SettingPage(R.string.set_airplane, R.drawable.ph_airplane, "airplane flight mode offline modo aviao voo", Settings.ACTION_AIRPLANE_MODE_SETTINGS),
        SettingPage(R.string.set_nfc, R.drawable.ph_contactless_payment, "nfc contactless payment tap pay pagamento por aproximacao", Settings.ACTION_NFC_SETTINGS),
        SettingPage(R.string.set_cast, R.drawable.ph_screencast, "cast screen mirror smart view tv espelhar transmitir televisao", Settings.ACTION_CAST_SETTINGS),
        SettingPage(R.string.set_vpn, R.drawable.ph_lock_key, "vpn private network rede privada", Settings.ACTION_VPN_SETTINGS),
        SettingPage(R.string.set_display, R.drawable.ph_sun, "display brightness screen dark mode font size timeout refresh rate tela brilho modo escuro fonte tamanho da fonte", Settings.ACTION_DISPLAY_SETTINGS),
        SettingPage(R.string.set_night_light, R.drawable.ph_moon_stars, "night light eye comfort blue light filter luz noturna conforto visual luz azul", Settings.ACTION_NIGHT_DISPLAY_SETTINGS),
        SettingPage(R.string.set_sound, R.drawable.ph_speaker_high, "sound volume ringtone vibration vibrate silent som volume toque vibracao vibrar silencioso", Settings.ACTION_SOUND_SETTINGS),
        SettingPage(R.string.set_dnd, R.drawable.ph_moon, "do not disturb dnd quiet silence nao perturbe silencio", Settings.ACTION_ZEN_MODE_PRIORITY_SETTINGS),
        SettingPage(R.string.set_battery, R.drawable.ph_battery_full, "battery power usage charging bateria energia carregamento", Intent.ACTION_POWER_USAGE_SUMMARY),
        SettingPage(R.string.set_battery_saver, R.drawable.ph_battery_full, "battery saver power saving low power economia de bateria economia de energia", Settings.ACTION_BATTERY_SAVER_SETTINGS),
        SettingPage(R.string.set_storage, R.drawable.ph_hard_drives, "storage space memory free up files armazenamento espaco memoria liberar arquivos", Settings.ACTION_INTERNAL_STORAGE_SETTINGS),
        SettingPage(R.string.set_apps, R.drawable.ph_squares_four, "apps applications manage permissions aplicativos permissoes", Settings.ACTION_APPLICATION_SETTINGS),
        SettingPage(R.string.set_default_apps, R.drawable.ph_app_window, "default apps browser phone sms apps padrao navegador", Settings.ACTION_MANAGE_DEFAULT_APPS_SETTINGS),
        SettingPage(R.string.set_home_app, R.drawable.ph_house, "home app launcher default tela inicial launcher padrao", Settings.ACTION_HOME_SETTINGS),
        SettingPage(R.string.set_location, R.drawable.ph_map_pin, "location gps position localizacao gps", Settings.ACTION_LOCATION_SOURCE_SETTINGS),
        SettingPage(R.string.set_security, R.drawable.ph_shield_check, "security lock screen fingerprint face password pin seguranca bloqueio digital rosto senha", Settings.ACTION_SECURITY_SETTINGS),
        SettingPage(R.string.set_privacy, R.drawable.ph_lock, "privacy permissions privacidade permissoes", Settings.ACTION_PRIVACY_SETTINGS),
        SettingPage(R.string.set_accounts, R.drawable.ph_user_circle, "accounts sync google backup contas sincronizacao", Settings.ACTION_SYNC_SETTINGS),
        SettingPage(R.string.set_accessibility, R.drawable.ph_person_simple_circle, "accessibility talkback magnification vision hearing acessibilidade ampliacao visao audicao", Settings.ACTION_ACCESSIBILITY_SETTINGS),
        SettingPage(R.string.set_language, R.drawable.ph_translate, "language region locale idioma regiao lingua", Settings.ACTION_LOCALE_SETTINGS),
        SettingPage(R.string.set_keyboard, R.drawable.ph_keyboard, "keyboard input typing teclado digitacao", Settings.ACTION_INPUT_METHOD_SETTINGS),
        SettingPage(R.string.set_date_time, R.drawable.ph_calendar, "date time clock time zone data hora relogio fuso horario", Settings.ACTION_DATE_SETTINGS),
        SettingPage(R.string.set_printing, R.drawable.ph_printer, "print printer printing impressora impressao imprimir", Settings.ACTION_PRINT_SETTINGS),
        SettingPage(R.string.set_developer, R.drawable.ph_code, "developer options usb debugging animations opcoes do desenvolvedor depuracao usb animacoes", Settings.ACTION_APPLICATION_DEVELOPMENT_SETTINGS),
        SettingPage(R.string.set_about, R.drawable.ph_device_mobile, "about phone model version software update sobre o telefone modelo versao atualizacao", Settings.ACTION_DEVICE_INFO_SETTINGS),
        SettingPage(R.string.set_all, R.drawable.ph_settings, "settings preferences configuracoes ajustes", Settings.ACTION_SETTINGS),
        SettingPage(R.string.set_canyon, R.drawable.ph_settings, "canyon launcher home screen grid icons layout transparency glass tela inicial grade icones transparencia vidro", null, SettingsActivity::class.java),
        SettingPage(R.string.set_wallpaper, R.drawable.ph_wallpaper, "wallpaper background photo papel de parede fundo foto", null, WallpaperActivity::class.java),
    )
}
