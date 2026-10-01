# RapidESP

Control the GPIO pins on an ESP32 board over your network using simple HTTP requests. You don't need to write any code or set up a per-board config. Flash the binary for your chip, join it to Wi-Fi, and then:

- ask the board which pins it has and what each one can do
- set pins high or low
- read pin state

It's handy for quick automation, testing hardware, or driving relays and LEDs from scripts, Home Assistant, Node-RED, and the like.

---

## Supported boards

Download the file for your board from the latest [release](../../releases). Each file is named `gpio-http-<board>-<version>.bin`.

| Board | File | Chip |
|---|---|---|
| ESP32 DevKit (most generic ESP32 boards) | `gpio-http-esp32-devkit-…bin` | ESP32 |
| ESP32-S3 DevKitC | `gpio-http-esp32s3-devkitc-…bin` | ESP32-S3 |
| ESP32-C3 DevKitM | `gpio-http-esp32c3-devkitm-…bin` | ESP32-C3 |
| ESP32-C6 DevKitC | `gpio-http-esp32c6-devkitc-…bin` | ESP32-C6 |
| Lolin S2 Mini | `gpio-http-lolin-s2-mini-…bin` | ESP32-S2 |

The firmware is built per **chip**, not per board. If your board isn't listed, pick the file that matches its chip. It will usually work, and you can mark any pins your board uses for something else as reserved (see [Board profile](#board-profile)).

---

## Flashing

### Option 1: ESPHome Web (easiest)

You need Chrome or Edge on a desktop. Firefox and Safari don't support Web Serial.

1. Connect the board via USB.
2. Go to <https://web.esphome.io> and click **Connect**. Choose your board's serial port.
3. Click **Install**. Choose the `.bin` file you downloaded, then click **Install** again.
4. When it's finished, press the board's **RST** button, or unplug it and plug it back in.

Don't use "Prepare for first use". That option installs ESPHome, not this firmware.

**If the board doesn't show up or won't connect** (common on S2, S3, C3 and C6 boards that use native USB), put it into download mode:

1. Hold the **BOOT** button. On the Lolin S2 Mini it's labelled **0**.
2. Press and release **RST**.
3. Release **BOOT**.
4. Connect again.

### Option 2: esptool (command line)

```sh
pip install esptool
esptool.py write_flash 0x0 gpio-http-<board>-<version>.bin
```

The `.bin` is a complete image, so it always goes at offset `0x0`.

---

## First-time setup

1. After flashing, the board creates a Wi-Fi network called **`gpio-http-xxxxxx`**.
2. Join it from your phone or laptop. A setup page should open. If it doesn't, browse to `192.168.4.1`.
3. Click **Configure WiFi** and pick your network. Then enter:
   - your Wi-Fi password
   - an **API token**, which is any password you like and is used to protect the API. Leaving it blank means anyone on your network can control the pins.
4. Save. The board restarts and joins your network.

The setup network closes after 5 minutes if nothing is configured, and the board then restarts and tries again.

**Finding the board's IP address:** check your router's client list, or open a serial monitor at 115200 baud. On boot the board prints its address, for example:

```
gpio-http v1.0.0 on ESP32-S3 — http://192.168.1.50/api/gpio
```

Giving it a DHCP reservation in your router is a good idea so the address doesn't change.

**Changing Wi-Fi later:** if the board can't reach its saved network, the setup network comes back automatically. To force it, erase the flash with `esptool.py erase_flash` and reflash. Erasing also clears the API token and board profile.

---

## Using the API

All examples assume the board is at `192.168.1.50` and the token is `mytoken`. If you didn't set a token, leave out the `Authorization` header.

### List all pins

```sh
curl http://192.168.1.50/api/gpio -H "Authorization: Bearer mytoken"
```

```json
{
  "chip": "ESP32-S3",
  "firmware": "v1.0.0",
  "pins": [
    {
      "pin": 4,
      "label": "Relay 1",
      "caps": ["input", "output", "adc1", "touch", "rtc"],
      "strapping": false,
      "reserved": false,
      "mode": "output",
      "level": 1
    }
  ]
}
```

What the fields mean:

| Field | Meaning |
|---|---|
| `caps` | What the pin can do: `input`, `output`, `adc1`/`adc2` (analog), `touch`, `rtc` (can wake the chip from deep sleep) |
| `strapping` | The pin affects how the chip boots. Usable, but anything connected to it can stop the board booting. |
| `reserved` | The pin is blocked from use, usually because it's wired to flash memory, USB or serial. |
| `mode` | `unset`, `input`, `input_pullup`, `input_pulldown` or `output` |
| `level` | `1` (high), `0` (low), or `null` if the pin hasn't been configured |

### Read one pin

```sh
curl http://192.168.1.50/api/gpio/4 -H "Authorization: Bearer mytoken"
```

### Set a pin high or low

```sh
curl -X POST http://192.168.1.50/api/gpio \
  -H "Authorization: Bearer mytoken" \
  -H "Content-Type: application/json" \
  -d '{"pin": 4, "level": "high"}'
```

`level` accepts `"high"`/`"low"`, `1`/`0`, or `true`/`false`. Setting a level automatically switches the pin to output mode.

### Set a pin as an input

```sh
curl -X POST http://192.168.1.50/api/gpio \
  -H "Authorization: Bearer mytoken" \
  -H "Content-Type: application/json" \
  -d '{"pin": 5, "mode": "input_pullup"}'
```

The modes are `input`, `input_pullup`, `input_pulldown` and `output`.

### Errors

Errors come back as `{"error": "..."}` with one of these status codes:

| Code | Meaning |
|---|---|
| 400 | Bad request, such as invalid JSON, an unknown mode or a bad level |
| 401 | Missing or wrong token |
| 404 | No such pin, or no such endpoint |
| 409 | The pin is reserved, or it's input-only and you tried to drive it |

---

## Board profile

The profile lets you adjust the default pin list for your specific board:

- **reserve** pins your board uses internally, such as an onboard LED or a display
- **release** pins the firmware reserved by default but your board doesn't use
- **label** pins with friendly names

```sh
curl -X PUT http://192.168.1.50/api/profile \
  -H "Authorization: Bearer mytoken" \
  -H "Content-Type: application/json" \
  -d '{"reserve": [21], "release": [43, 44], "labels": {"4": "Relay 1", "5": "Door sensor"}}'
```

The profile is saved on the board and survives restarts. Use `GET /api/profile` to see the current one. Sending `{}` resets to the defaults.

**Releasing a reserved pin is at your own risk.** Some reserved pins are connected to the flash memory, and driving them will crash the board. Serial and USB pins are safer to release, but doing so breaks serial logging or USB on boards that use them.

---

## Safety notes

- ESP32 pins are **3.3 V only** and can only supply a small current. Use a transistor, MOSFET or relay module to switch anything bigger than an LED. Don't connect a bare relay coil or motor directly to a pin.
- Pins reset to an unconfigured state when the board restarts, so outputs don't stay on across reboots.
- Avoid wiring anything to **strapping** pins if you can. Something pulling one of them high or low at power-up can stop the board booting.
- The API runs over plain HTTP. Set a token, keep the board on a trusted network, and don't expose it to the internet.

---

## Troubleshooting

| Problem | Try |
|---|---|
| ESPHome Web can't connect | Use Chrome or Edge, try a different USB cable (some are charge-only), and use download mode (see [Flashing](#flashing)) |
| Board doesn't restart after flashing | Press RST. Native-USB boards often need this. |
| No `gpio-http-…` Wi-Fi network | Check the serial monitor. If the board connected to a saved network, it won't open setup mode. |
| `401 unauthorised` | Check the header is exactly `Authorization: Bearer <token>` |
| Pin shows `reserved` but your board doesn't use it | Release it via the [board profile](#board-profile) |
| `adc2` pin readings are unreliable (ESP32) | ADC2 can't be used while Wi-Fi is active on the original ESP32. Use an `adc1` pin. |

---

## Development notes

### Repo layout

```
firmware/gpio-http/gpio-http.ino   # the firmware (single sketch)
.github/workflows/release.yml      # builds and attaches binaries on release
```

### Dependencies

- [arduino-esp32](https://github.com/espressif/arduino-esp32) core 3.x, which is based on ESP-IDF 5.x
- [ArduinoJson](https://arduinojson.org/) 7.x
- [WiFiManager](https://github.com/tzapu/WiFiManager) (tzapu)

`WebServer` and `Preferences` are built into the core.

### Building locally

```sh
arduino-cli core update-index --additional-urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
arduino-cli core install esp32:esp32 --additional-urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
arduino-cli lib install ArduinoJson WiFiManager

arduino-cli compile --fqbn esp32:esp32:esp32s3 --output-dir build firmware/gpio-http
arduino-cli upload  --fqbn esp32:esp32:esp32s3 -p /dev/cu.usbmodem* firmware/gpio-http
```

Local builds report their firmware version as `dev`. The release workflow writes `version.h` with the release tag before compiling.

### Releases

Publishing a GitHub release triggers `.github/workflows/release.yml`. It compiles each board in the matrix and uploads `build/gpio-http.ino.merged.bin` to the release, renamed to `gpio-http-<board>-<tag>.bin`.

The core version is currently unpinned, so releases use the latest 3.x core. Pin it (`esp32:esp32@<version>`) once you've tested a version.

### Adding a board

If the board uses an already-supported chip, add an entry to the workflow matrix with a `name` and its `fqbn`. To list available FQBNs, run `arduino-cli board listall esp32`.

### Adding a chip

1. Add a `#elif CONFIG_IDF_TARGET_<CHIP>` block to the tables near the top of the sketch:
   - `STRAPPING`: strapping pins, taken from the chip datasheet
   - `RESERVED`: default reservations, i.e. SPI flash, native USB and UART0
   - `RESERVED_PSRAM`: pins only reserved when PSRAM is detected

   Terminate each list with `NO_PIN`. Unsupported targets hit an `#error` deliberately.
2. Add a board for the chip to the workflow matrix.

### How pin capabilities are determined

| Capability | Source |
|---|---|
| Pin exists | `SOC_GPIO_VALID_GPIO_MASK` |
| Output | `SOC_GPIO_VALID_OUTPUT_GPIO_MASK` |
| ADC | `adc_oneshot_io_to_channel()`, which also gives the unit |
| Touch | `digitalPinToTouchChannel()`, guarded by `SOC_TOUCH_SENSOR_SUPPORTED` |
| RTC | `rtc_gpio_is_valid_gpio()`, guarded by `SOC_RTCIO_PIN_COUNT > 0` |
| Strapping and reserved | Hand-maintained per-chip tables (see [Adding a chip](#adding-a-chip)) |

Output pins are configured as `GPIO_MODE_INPUT_OUTPUT` so that `digitalRead()` returns the driven level.

### Known limitations and open items

- The default reservation tables are best-effort for common modules. Boards with unusual flash or PSRAM wiring need a profile.
- On the ESP32-S3, GPIO33–37 are reserved whenever PSRAM is detected, but only octal-PSRAM modules use them.
- `digitalPinToTouchChannel()` may change with the reworked touch API in newer 3.x cores.
- Pin modes and levels are not persisted across reboots.
- There's no mDNS, no OTA and no HTTPS.
