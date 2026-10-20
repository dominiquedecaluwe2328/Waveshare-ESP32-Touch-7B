# Waveshare ESP32-S3-Touch-LCD-7B – ESPHome base config

A working ESPHome + LVGL starting point for the **Waveshare ESP32-S3-Touch-LCD-7B** (7", 1024×600, GT911 touch, CH32V003 IO expander).

It contains only what you need to get a stable screen. You add your own dashboard on top:

- RGB display (`rpi_dpi_rgb`) with timings that give a stable, flicker-free image
- dimmable backlight through the IO expander PWM output (shows up as a light in Home Assistant)
- GT911 touch with correct calibration
- one LVGL test page that shows uptime, free RAM and a button to test touch

Tested with ESPHome 2026.9.1 (ESP-IDF framework).

## Two fixes for a "flaky" 7B

If your board sometimes boots with a black or very dim screen, or touch only works on some boots, these two problems are the likely cause. Both fixes are in this config.

**1. Backlight stays off or very dim after boot.** On this board EXIO2 enables the backlight and the IO expander PWM sets the brightness. If EXIO2 is linked to the PWM output as a `power_supply`, the backlight sometimes doesn't come up: the screen content is there (you can see it faintly with a flashlight), and changing the brightness does nothing.
Fix: EXIO2 is a plain `gpio` switch, and the backlight is switched on again 1 s after boot.

**2. Touch fails on a cold boot.** The log shows `touchscreen is marked FAILED: Communication failed`. When the first attempt fails, ESPHome's GT911 driver switches to the secondary address `0x14` and stays there. On this board the chip is at `0x5D`, so it never recovers.
Fix: every 3 s, if touch has failed, an interval sets the address back to `0x5D` and runs the setup again (up to 20 tries).

## Pinout (from Waveshare's documentation and this config)

| Function | Pin |
|---|---|
| I2C SDA / SCL | GPIO8 / GPIO9 |
| IO expander (CH32V003) | I2C 0x24 |
| Touch GT911 | I2C 0x5D, INT GPIO4, reset EXIO1 |
| Backlight enable | EXIO2 |
| LCD reset | EXIO3 |
| SD card CS | EXIO4 |
| USB / CAN select | EXIO5 |
| LCD VDD enable | EXIO6 |
| RGB data | R 1,2,42,41,40 · G 39,0,45,48,47,21 · B 14,38,18,17,10 |
| DE / PCLK / HSYNC / VSYNC | 5 / 7 / 46 / 3 |

## Usage

1. Copy `waveshare-esp32-s3-touch-lcd-7b.yaml` into your ESPHome config folder.
2. Add `wifi_ssid`, `wifi_password` and `api_encryption_key` to your `secrets.yaml` (see `secrets.yaml.example`). You can generate an API key on the [ESPHome API docs page](https://esphome.io/components/api/).
3. Flash the first time over USB. After that, updates go over wifi (OTA).
4. If your flash partition is too small for OTA after flashing a different firmware, flash the factory image over USB once.

## Battery

- The board has a PH2.0 connector for a single 3.7 V Li-ion cell and a CS8501 charger. Charging needs no configuration.
- **Check the polarity first.** Many PH2.0 batteries are wired the other way round. Compare red/black with the + and − on the board before you plug it in.
- **The battery only charges when the board is powered through the UART USB-C port.** On the USB port the board runs fine, but the CHARGE LED stays off. LEDs: POWER, CHARGE (charging), DONE (full).
- Battery voltage can be read through the ADC of the IO expander:

```yaml
sensor:
  - platform: waveshare_io_ch32v003
    waveshare_io_ch32v003_id: io_ex
    name: Battery voltage
    reference_voltage: 9.9   # 3.3 V x 3:1 divider; calibrate against a multimeter
    update_interval: 30s
```

## Tips for building a dashboard on this board

- Keep `buffer_size: 100%` (in PSRAM). A smaller buffer goes to internal RAM and leaves too little for wifi.
- Watch the free internal RAM, which the test page shows. On a big dashboard, stay above roughly 100 kB.
- Every Home Assistant sensor you import costs RAM and API traffic. Only import what a page actually shows.
- Avoid `lvgl.pause` with `show_snow` as an idle screen. A full-screen black object plus backlight off is more reliable.
- Album art and other images: let the server send small images over plain HTTP. On the ESP, decoding large JPEGs and doing HTTPS handshakes freezes the UI for a moment.

## License

MIT – use it, change it, share it.
