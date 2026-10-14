# OpenSprinkler firmware for Sonoff 4CH Pro R3

This is a fork of [OpenSprinkler Firmware](https://github.com/OpenSprinkler/OpenSprinkler-Firmware) with support for the **Sonoff 4CH Pro R3** (ESP8285-based 4-channel relay board).

Genuine OpenSprinkler hardware drives stations through I2C IO expanders (PCA9555/PCF8574). The Sonoff has no expanders — its four relays are wired to GPIO pins. This fork drives those pins directly. All Sonoff-specific code is behind the `SONOFF_4CH_PRO_R3` build flag, so genuine OS3.x ESP8266 builds (`os3x_esp8266`) are unchanged.

Upstream documentation: [OpenSprinkler Firmware docs](https://opensprinkler.github.io/OpenSprinkler-Firmware/)

---

## What you get

- Four physical relays, shown as **stations 1–4** in the web UI (stations 5–8 are hidden)
- **Physical buttons** that toggle the matching relay (same runtime queue as the web UI Manual Run)
- Relays stay on for **18 hours**, or until you turn them off from a button or the web UI
- **Factory reset** by holding physical Button 4 at boot (reboots into AP mode)
- Optional rain/flow sensor on GPIO16

---

## Hardware pin mapping

| Function | GPIO | Notes |
|----------|------|-------|
| Relay 1 | GPIO12 | Active HIGH |
| Relay 2 | GPIO5 | Active HIGH (ESP8266 default I2C SCL — remapped away in software) |
| Relay 3 | GPIO4 | Active HIGH (ESP8266 default I2C SDA — remapped away in software) |
| Relay 4 | GPIO15 | Active HIGH |
| LED | GPIO13 | Inverted (LOW = ON) |
| Physical Button 1 | GPIO0 | Active LOW. **Hold at boot to enter flash mode.** Mapped in software to logical Button 4 |
| Physical Button 2 | GPIO9 | Active LOW |
| Physical Button 3 | GPIO10 | Active LOW |
| Physical Button 4 | GPIO14 | Active LOW. Mapped in software to logical Button 1. Used for factory reset |
| Sensor 1 | GPIO16 | INPUT_PULLUP (rain/flow). Unconnected reads HIGH (“no rain”), so scheduled programs run |

Buttons are independent GPIOs (not a matrix), active LOW with internal pull-ups.

**Software button swap:** physical Button 1 (GPIO0) is logical Button 4, and physical Button 4 (GPIO14) is logical Button 1. That keeps GPIO0 from being treated as the primary control button, so you do not accidentally hold it at boot and enter flash mode.

---

## Build and flash

You need [PlatformIO](https://platformio.org/) and a **3.3 V USB-to-TTL serial adapter**.

### Build

```bash
pio run -e sonoff_4ch_pro_r3
```

Genuine OpenSprinkler ESP8266 hardware (no Sonoff changes):

```bash
pio run -e os3x_esp8266
```

### Flash

1. Connect the serial adapter (3.3 V TX/RX/GND; do **not** use 5 V on the ESP pins).
2. Hold **physical Button 1** (GPIO0) while powering the board to enter flash mode.
3. Upload:

```bash
pio run -e sonoff_4ch_pro_r3 --target upload
```

The PlatformIO environment is `[env:sonoff_4ch_pro_r3]` in `platformio.ini` (`board = esp8285`, flag `-DSONOFF_4CH_PRO_R3`). Serial upload and monitor speed are 115200.

Hardware wiring, flash-mode entry, and R3-specific notes (including the R21-to-ground alternative) are documented in the [Tasmota Sonoff 4CH Pro R3 flashing guide](https://tasmota.github.io/docs/devices/Sonoff-4CH-Pro/#sonoff-4ch-pro-r3). Follow that for the serial connection; use the PlatformIO command above instead of Tasmota’s firmware image.

After flashing, join the device’s AP (or your configured Wi-Fi) and use the OpenSprinkler web UI as usual. Only four stations appear.

---

## Using the device

### Relays / stations

Turn stations on or off from the web UI **Manual Run**, or press the matching physical button. Button presses enqueue and dequeue stations on the same path as the web UI test button.

### Physical buttons

Each press is debounced (200 ms) so a mechanical bounce does not toggle twice.

| Physical button | GPIO | Relay |
|-----------------|------|-------|
| Button 1 | GPIO0 | Relay 4 (logical Button 4) |
| Button 2 | GPIO9 | Relay 2 |
| Button 3 | GPIO10 | Relay 3 |
| Button 4 | GPIO14 | Relay 1 (logical Button 1) |

Do not hold physical Button 1 at power-on unless you intend to flash firmware.

### Factory reset

The upstream boot reset uses a button matrix and LCD UI that this board does not have. Use this instead:

1. Power off the Sonoff.
2. Hold **physical Button 4** (GPIO14, the button on the blue terminal block).
3. Power on while holding. **Relay 4** turns on as feedback that the hold is detected.
4. Keep holding for about **3 seconds**.
5. The firmware runs `pre_factory_reset()`, then reboots into **AP mode**. Settings are wiped.

If you release the button too early, reset is cancelled and the board boots normally.

---

## Why the stock firmware does not work on this board

These are the hardware mismatches this fork works around:

- **Relays 2 and 3** sit on GPIO5 and GPIO4, the ESP8266 default I2C SCL/SDA pins. The `Wire` library leaves those pins in `INPUT_PULLUP` after each transaction, which cannot drive relay coils. Relays blink or stay off whenever the firmware talks I2C (including SSD1306 display updates that still run even with no OLED).
- **Physical buttons** are independent GPIOs, not the OpenSprinkler button matrix, so `button_read()` never reports presses.
- **GPIO14** (physical Button 4) is taken as SPI CLK by `SPI.begin()`, so it looks constantly pressed unless reclaimed as `INPUT_PULLUP`.

### What this firmware does

1. **Remap I2C** — `Wire.begin()` uses GPIO2 (SDA) and GPIO13 (SCL) so I2C never touches GPIO4/GPIO5.
2. **Drive relays on GPIO** — `apply_all_station_bits()` and `set_station_bit()` write the four relay pins directly (no I2C expanders). Each write does `pinMode(OUTPUT)` then `digitalWrite()` so the pin can source coil current.
3. **Reclaim pins every loop** — at the end of `do_loop()` (hundreds of times per second), all relay pins are forced back to `OUTPUT` and driven to the current station bits. That recovers from any code that temporarily reconfigures a pin.
4. **Read buttons on GPIO** — `ui_state_machine()` skips `button_read()`, uses `digitalRead()`, and applies 200 ms debounce.
5. **Show only 4 stations** — `nstations` is capped at 4 in `begin()`, `iopts_load()`, and `iopts_save()`. `factory_reset()` also writes `dis=1` for stations 5–8 so placeholders stay disabled.
6. **Factory reset on Button 4** — `options_setup()` reads GPIO14 directly (3 second hold) instead of `button_read()`.
7. **Skip I2C RTC and expander auto-detect** — there is no RTC or PCA9555/PCF8574 on the Sonoff. Pin assignments stay the static map in `defines.cpp`.

---

## Source files involved

| File | Role |
|------|------|
| `platformio.ini` | `sonoff_4ch_pro_r3` environment |
| `defines.h` | `PIN_BUTTON_4` |
| `defines.cpp` | Static Sonoff pin map |
| `OpenSprinkler.cpp` | I2C remap, GPIO relays, station cap, factory-reset defaults and boot button, skip RTC/auto-detect, button mapping |
| `OpenSprinkler.h` | Pin externs (already present) |
| `main.cpp` | Per-loop pin reclamation, GPIO buttons, debounce, queue on/off |
| `I2CRTC.cpp` | Skip `Wire.begin()` on Sonoff |

Deeper implementation notes (including I2C dropout root cause) are in [`SONOFF_4CH_PRO_R3_SUPPORT.md`](SONOFF_4CH_PRO_R3_SUPPORT.md).
