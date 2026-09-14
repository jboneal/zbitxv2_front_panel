# zBitx Front Panel v2 — Code Evaluation

- **Created (workspace/UTC):** 2026-09-14 22:12 UTC
- **Created (local EST/EDT):** 2026-09-14 18:12 EDT
- **Commit evaluated:** `63caacf` on `main` ("added wifi instructions and set the version", 2026-04-27)
- **Method:** Full read of every source file (about 4,500 lines). No compile or hardware test was performed. There is no RP2040/Arduino toolchain in this container, and the tree cannot compile anyway (see finding 1). Every finding below was verified by reading the code directly; where behaviour on hardware is inferred rather than observed, it is labelled **Likely** instead of **Confirmed**.

---

## 1. Verdict

This is working hobbyist firmware that has clearly been run on real hardware, but the repository as committed is **not buildable**, has **one real memory-safety bug** reachable from the network, and **two silent watchdog/robustness defeats** that undermine the "make tcp/ip robust" work in the April 2026 commits. The dual-core design has no synchronisation at all. Documentation (README, CLAUDE.md, Read_Me.ino) describes a build setup that no longer matches the files in the tree.

Severity summary:

| Severity | Count | Examples |
|---|---|---|
| Blocker (won't build) | 1 | Missing `fonts/` directory |
| High (crash / corruption) | 4 | Field value overflow from radio, FT8 buffer overflows, NULL deref in field_select, TCP read buffer off-by-one |
| Medium (wrong behaviour) | 10 | Core-1 watchdog defeated, EEPROM compare skips every other byte, divide-by-zero in SWR, logbook shows wrong row's frequency, unsynchronised cross-core drawing |
| Low (hygiene / dead code / docs) | 12 | Stale CLAUDE.md, unused WiFi credential store, debug spam in draw loops, header guard typo |

---

## 2. Blocker: repository does not build as committed

**Confirmed.** `tft_ili9488.cpp:80-82` does:

```c
#include "fonts/glcdfont.c"
#include "fonts/Font16.c"
#include "fonts/Font32rle.c"
```

There is no `fonts/` directory in the tree, and `git log --all -- fonts` shows it has never been committed. These are the three font source files from the TFT_eSPI library's `Fonts/` folder. The author's local sketch folder must contain them, but a fresh clone cannot compile. The committed `.uf2` is the only way to get a running image from this repo.

**Fix:** commit the three files (they are FreeBSD-licensed in TFT_eSPI, so include the license header), or vendor them into `free_font.h` which already exists but is not used by the driver.

---

## 3. High-severity findings

### 3.1 Any radio message longer than 127 characters overflows the field struct
**Confirmed.** `fields.ino:210` copies the incoming value with `strcpy(f->value, value)`. `value` comes from `cmd_value[1000]` in the tokenizer, but `field::value` is `char[128]` (`zbitx.h:93`). A length check existed here and is commented out on the line above. The overflow runs into `selection[128]`, then the `draw` function pointer, then `data`. A corrupted `draw` pointer is called on the next redraw.

The radio is a trusted peer today, but this also breaks on any long console line, long TEXT string, or a future protocol field. The WF and console labels are special-cased and avoid this path, which is probably why it hasn't bitten yet.

**Fix:** `strncpy(f->value, value, FIELD_TEXT_MAX_LENGTH - 1); f->value[FIELD_TEXT_MAX_LENGTH-1] = 0;` or restore the length guard.

### 3.2 FT8 parser has two unchecked `strcpy` calls
**Confirmed.** `ft8.cpp:59` does `strcpy(buff, msg)` into `buff[100]`, and `ft8.cpp:83` does `strcpy(m->data, msg)` into `data[100]`. The only length check (`ft8.cpp:80`) tests the substring after the tilde, not `msg` itself. A decoded FT8 line is typically 40-60 characters, so this works by margin, but the incoming value can be up to 999 bytes.

### 3.3 NULL dereference in `field_select`
**Confirmed.** `fields.ino:284`:

```c
else if (!strcmp(f_selected->label, "SAVE")){
```

`f_selected` is set to NULL after every button press (`fields.ino:311`) and on FINISH. Tap any button, then tap any other field: this line dereferences NULL. It was almost certainly meant to be `f->label` (the field just tapped), which would also make "SAVE clears the logbook" actually work. As written, SAVE never triggers `logbook_init()`.

**Likely** reason it hasn't crashed in testing: on the RP2040 the bootrom is mapped at address 0, so reading `NULL->label` reads ROM instead of faulting. I have not verified that on hardware; it is my reading of the RP2040 memory map. It is still undefined behaviour and will break under a different compiler optimisation level.

### 3.4 TCP read buffer off-by-one
**Confirmed.** `zbitx_front_panel_v2.ino:547` sets `bytes_to_read = sizeof(buff)` (1000), then line 554 writes `buff[actually_read] = 0`. When 1000 bytes are available, that writes `buff[1000]`, one past the end of the array declared at line 23. Should be `sizeof(buff) - 1`.

---

## 4. Medium-severity findings

### 4.1 Core-1 watchdog is defeated by core 0
**Confirmed.** `core1_check()` restarts core 1 if `core1_time` is more than 10 s stale. Core 1 refreshes it at line 454. But **core 0 also refreshes it** at `zbitx_front_panel_v2.ino:562` on every pass through `loop()` once WiFi is up. So a hung display core is never detected while WiFi is connected, which is the only time it matters. Delete line 562.

### 4.2 EEPROM change-detection reads every other byte
**Confirmed.** `storage.cpp:53`:

```c
for (i = 0; i < sizeof(buff); i++)
    buff[i] = EEPROM.read(i++);
```

`i` is incremented twice per iteration, so odd bytes of `buff` are never filled. The `memcmp` that follows compares garbage, so the "skip write if unchanged" optimisation fails and every `block_write()` performs a flash erase/write cycle. Not fatal (writes are rare) but it is exactly the wear the comment "don't do this too often" is trying to avoid.

### 4.3 SWR calculation divides by zero at idle
**Confirmed.** `zbitx_front_panel_v2.ino:304`: `vswr = (10*(vfwd + vref))/(vfwd-vref)`. With no RF, `vfwd == vref` is the normal case. **Likely** no fault on RP2040 because the SDK's integer divide goes through the hardware divider, which returns a garbage quotient rather than trapping; the garbage then goes to the radio as `vswr` every 200 ms. Guard with `if (vfwd > vref)`.

### 4.4 Forward-power average is corrupted by the display transform
**Confirmed.** `zbitx_front_panel_v2.ino:307` does `vfwd = 3 + vfwd/2` **on the state variable** after computing the moving average. The next call averages this already-transformed value again. The "3 +" offset also means `vfwd` never reaches 0, so the panel reports a small non-zero `power` to the radio at idle. Keep the running average in a separate variable and derive the display value from it.

Also: `zbitx.h:34-35` define `VFWD A1` and `VREF A0`, but the code reads forward from A0 and reflected from A1 (lines 285-286). The macros are unused, so one of the two is wrong. I cannot tell which from the code.

### 4.5 No cross-core synchronisation
**Confirmed** by reading; **Likely** cause of intermittent glitches rather than crashes. `fields.ino:2` includes `pico/sync.h` but no mutex, spinlock, or critical section is used anywhere. Specifically:

- Core 0 (`field_set`) `strcpy`s into `f->value` while core 1 reads it for drawing. Torn strings.
- Core 0 receiving `MODE` calls `field_set_panel` (`fields.ino:209`), which calls `keyboard_hide`, which calls `field_draw_all(true)` (`text_field.cpp:179`) — a full-screen SPI redraw **from core 0** while core 1 may be mid-transaction on the same SPI1 bus. The commit "stable with core1 doing all the display access" is not true after that path.
- Core 0 `waterfall_update` writes the 48 KB waterfall buffer while core 1 `pushImage`s it.
- Both cores call `block_dump()` / `block_read()` on the shared `block` struct at boot without ordering.
- `wheel_move` is modified in an ISR and reset in `ui_slice` without `volatile` or atomics.

Minimum fix: route all `field_set` calls through a queue drained on core 1 (the unused `queue.cpp` is a hint that this was once the plan), and keep every SPI call on core 1.

### 4.6 Dead TCP connections are never detected
**Confirmed.** `last_rx_ms` and `SERVER_RX_TIMEOUT_MS` (lines 44-45) are declared and never used. If the radio stops sending without closing the socket, `client.connected()` can stay true indefinitely and the panel keeps sending `?\n` every 200 ms into a dead socket. The commit message "code changes to make tcp/ip robust (not really working)" matches this: the timeout was planned but not wired in.

### 4.7 Logbook draws the wrong row's frequency
**Confirmed.** `logbook.cpp:189` uses `logbook[line].frequency` instead of `e->frequency`. Correct only while `log_top_index == 0`; once the list scrolls, each row shows the frequency of a different QSO.

### 4.8 FT8 cursor logic indexes with -1
**Confirmed.** In `ft8_move_cursor` (`ft8.cpp:108-121`), when `ft8_cursor == -1` and `by != 0`, the `if (by < 0)` / `else if` branches run after the `-1` case and evaluate `ft8_list[ft8_cursor]` with `ft8_cursor == -1`. Out-of-bounds read one struct before the array. Add `else` before the `if (by < 0)`.

Also `ft8_touched` (`ft8.cpp:221`) does not check that the tapped slot holds a message, so tapping blank space sends `FT8 \n` to the radio.

### 4.9 Text-editor visible-window scan reads one byte before the buffer
**Confirmed.** `text_field.cpp:190-198`: the do/while decrements `p` and only checks `p >= f->value` after reading `font_width2[*p]`. For any short string the loop exits with `p == f->value - 1` and returns it, so `text_draw` starts one byte early. **Likely** invisible in practice because the byte before `value` is the high byte of the `label` pointer, which is a non-printable control code the font skips. Also `*p` is a signed `char` used as an array index, so any byte over 127 indexes negative.

### 4.10 Console draws at `f->w + 2` instead of `f->x + 2`
**Confirmed.** `console.cpp:69`. Works only because the console field happens to have `x == w == 240`. Same signed-char index problem at `console.cpp:39`.

### 4.11 Waterfall heat map produces wrong colours above mid-scale
**Likely.** `waterfall.cpp:47` shifts to a value up to 0xF80, overflowing the 6-bit green field into red; line 52 uses `0x7E00` where green is `0x07E0`; line 62 computes `(160-v) << 6` with `v` up to 1020, which is a negative shift assigned to `uint16_t`. The comment says these values were found empirically over two days, so they may be intentionally compensating for byte order. I cannot verify without hardware, but line 62 is arithmetically wrong for strong signals regardless.

### 4.12 Logbook delete dialog posts MESSAGE to the radio
**Confirmed.** `logbook.cpp:27`: `field_set("MESSAGE", entry, true)` with `update_to_radio = true`, so the dialog text is transmitted as `MESSAGE <text>\n`. Almost certainly should be `false`. `qso_str[10]` on line 25 also overflows for any qso_id of 10 digits.

---

## 5. Low-severity and hygiene

1. **CLAUDE.md is stale.** It references `platformio.ini`, `package.json`, the `TFT_eSPI` and `Wire` libraries, and a function `on_request()`. None exist. The real TFT layer is the vendored `tft_ili9488.cpp`, the outgoing update function is `send_updates()`, and the only supported build is Arduino IDE. Any future AI-assisted work on this repo will be misled by it.
2. **Read_Me.ino / screen_gx.cpp comments** still instruct copying `platform.local.txt`, which is not in the repo and is not needed since TFT_eSPI was dropped.
3. **Saved WiFi credentials are never used.** `block.ap_list` is read, written (`wifi_save`, which is `static` and never called), and dumped, but `wifi_poll` hardcodes `"zbitx"/"zbitx12345"` at line 493. `WiFiMulti multi`, `temp_ssid`, `temp_key` are unused.
4. **`block_dump()` prints stored WiFi keys in plaintext over USB serial** at boot (`storage.cpp:87`). Low risk, but unnecessary.
5. **Debug output inside draw loops.** `ft8.cpp:151` and `logbook.cpp:185` `printf` on every redraw. Over USB CDC this can stall the display core when no host is reading.
6. **`smeter_draw` runs twice per frame** (`fields.ino:610` and `fields.ino:791`).
7. **Version strings disagree:** screen says "v4.00 2026-04-27" (line 444), serial banner says "4.01 2026/04/17" (line 516).
8. **`queue.cpp` is dead code** and buggy: `memset(p->data, 0, p->max_q+1)` clears bytes not ints, `zbitx.h` is included twice mid-file, and the full-queue test is off by one. It also costs 16 KB of RAM if anyone ever instantiates it.
9. **`fields_list.h` guard defines `FIELDS_H`** (line 2) instead of `FIELDS_LIST_H`, so the guard never engages. Harmless today because the header is included exactly once, but it defines a non-`static` array so a second include would be a link error.
10. **`storage.cpp:78` `#define Serial1 Serial`** mid-file, while `logbook.cpp:54` uses the real `Serial1` (UART on pins 16/17) which is never `begin()`-ed. Pick one debug channel.
11. **`field_get` collides with keyboard labels.** Radio labels "5", "6", "7", "9" resolve first to the keyboard key fields with the same label before being translated. Works by accident; a radio label "1"-"4" or "8" would overwrite a key cap.
12. **Unreachable cases** in `smeter_draw` (cases 6 and 7 with a loop bound of 6); unused variables `last_y`, `req_count`, `total`, `last_sent`, `next_update` in `loop1`.

---

## 6. What is good

- The **field abstraction** is a sensible design for a small touch UI: one static table, uniform draw/input dispatch, string-keyed protocol mapping. It is easy to add a control.
- The **protocol** is trivially debuggable (plain text, one line per update, `?\n` poll).
- **`tft_ili9488.cpp`** is a clean, well-commented driver: DMA row transfers with a paired RX drain channel, transparent glyph rendering with run-length bursts, clipping in `pushImage`. This is the best-engineered file in the repo.
- **Boot-to-bootloader on encoder press** (`reset_usb_boot`) is a nice field-serviceability touch.
- The **WIFI_CHANNEL_FIX.txt** write-up is clear, correct about the shared-radio channel problem, and user-friendly.

---

## 7. Recommended order of work

1. Commit the `fonts/` files so a clone builds.
2. Fix the four High items (3.1 through 3.4). Each is a one-to-three line change.
3. Delete `zbitx_front_panel_v2.ino:562` so the core-1 watchdog works.
4. Wire up `last_rx_ms` in `loop()` and drop the socket on timeout.
5. Move all `field_set` work off core 0 onto a queue drained by core 1; remove the `field_draw_all` call from the `keyboard_hide` path when invoked from core 0.
6. Fix the SWR divide and the `vfwd` feedback.
7. Rewrite CLAUDE.md and README to describe the actual build (Arduino IDE, earlephilhower core, vendored driver, no PlatformIO).
8. Strip the `printf` calls from draw routines.

---

## 8. Things I could not verify

- Whether the `0x7E00` heat-map constants are deliberate byte-order compensation (needs the display).
- Whether `EEPROM.commit()` on core 1 while core 0 runs from flash is safe under the arduino-pico core version in use (the core normally handles this, but the version is not pinned anywhere in the repo).
- Actual RAM headroom. Static allocation is roughly 48 KB waterfall + ~28 KB fields + ~11 KB FT8 + ~16 KB logbook + ~5 KB TFT buffers on a 264 KB part, so it should be fine, but I did not link it.
- Whether A0 or A1 is physically the forward-power sense (see 4.4).
