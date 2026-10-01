# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Front panel controller firmware for the [zBitx HF transceiver](https://github.com/afarhan/zbitx), running on a Raspberry Pi Pico (RP2040) with a 480×320 TFT touchscreen. It exchanges newline-delimited `LABEL VALUE` text with the radio application on the Pi. The transport is selected by `PANEL_LINK` in `zbitx.h`: USB CDC over the panel's CAT port (default; the WiFi radio is never started), or the original WiFi TCP client to `192.168.4.1:8081`. See README.md for why the GPIO UART cannot be used on this board.

## Build & Flash

Arduino IDE or `arduino-cli` with the `arduino-pico` core. No PlatformIO config exists and no external TFT library is used; the display driver is the vendored `tft_ili9488.cpp`. The driver `#include`s three font sources from a `fonts/` folder that is not committed (see README.md). Build target is `rpipico` for the USB link, `rpipicow` only for the WiFi build.

## Architecture

### Core UI Model — `struct field`
Everything on screen is a `struct field` (defined in `zbitx.h`). Fields have a `type` (BUTTON, NUMBER, SELECTION, TEXT, FREQ, WATERFALL, FT8, LOGBOOK, SMETER, etc.), pixel coordinates/size, a `label` (the key used in the radio protocol), and a `value` string.

- `fields_list.h` — static array `main_list[]` declaring every field; first `FIELDS_ALWAYS_ON` (20) fields are permanently visible.
- `fields.ino` — field lifecycle: init, lookup by label (`field_get`), selection, input dispatch, draw, and posting updates back to the radio (`field_post_to_radio`).
- `screen_gx.cpp` — low-level drawing primitives wrapping the vendored `TFT_ILI9488` driver.

### Radio Communication Protocol
Plain-text over TCP WiFi socket. Each update is one line: `LABEL VALUE\n`. The front panel parses incoming lines in `command_tokenize()` (in `zbitx_front_panel_v2.ino`) and calls `field_set()`. Outgoing updates are batched in `send_updates()` by scanning `field->update_to_radio`; `send_text()` writes to whichever link `PANEL_LINK` selects. All debug output goes through `Debug` (never `Serial` directly), because in USB mode `Serial` is the radio link.

### Dual-Core Loop
- **Core 0** (`loop()`): encoder reading, touchscreen polling, field input dispatch, WiFi/TCP management.
- **Core 1** (`loop1()`): display rendering (`field_draw_all`), waterfall updates.

### Persistent Storage
`storage.cpp` — EEPROM-backed `struct saved` (magic `0x00C0FFEE`) storing WiFi AP credentials (up to 5) and calibration data. Use `block_read()` / `block_write()`.

### Subsystems
| File | Purpose |
|---|---|
| `waterfall.cpp` | Scrolling FFT waterfall display |
| `ft8.cpp / ft8.h` | FT8 decode display and QSO logging |
| `logbook.cpp / logbook.h` | On-screen QSO logbook |
| `console.cpp / console.h` | Scrolling text console field |
| `text_field.cpp / text_field.h` | Editable text input with on-screen keyboard |
| `queue.cpp` | Simple fixed-size ring buffer (`struct Queue`, capacity 4000) |

## Key Conventions

- Fields are looked up by string label (`field_get("FREQ")`), not by index.
- `f_selected` (global in `fields.ino`) tracks the one currently active field.
- The `update_to_radio` flag on a field is set by `field_post_to_radio()` and cleared after transmission in `send_updates()`.
- `last_user_change` timestamp prevents radio updates from overwriting in-flight user edits (1 s debounce, except for `FIELD_TEXT`).
- Screen is 480×320; font sizes are 1 (small), 2 (normal), 4 (large) — corresponding width tables are `font_width2[]` and `font_width4[]`.
