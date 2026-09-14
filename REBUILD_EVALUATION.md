# zBitx V2 Rebuild — Combined Evaluation (Pi app + Front Panel)

- **Created (workspace/UTC):** 2026-09-14 23:12 UTC
- **Created (local EST/EDT):** 2026-09-14 19:12 EDT
- **Repos evaluated:**
  - `jboneal/zbitx2-rebuild` @ `f5d5276` (2026-05-29) — Raspberry Pi radio application (sbitx v4.001)
  - `jboneal/zbitx2-front-rebuild` @ `4bef03e` (2026-05-29) — Pico W front-panel firmware
- **Compared against:** `afarhan/zbitx` `main` @ `c671ac8` (2025-07-05) and `farhandev` @ `5cc4287`; `afarhan/sbitx` `farhan`; and the stock panel `jboneal/zbitxv2_front_panel` @ `63caacf` (evaluated separately in `CODE_EVALUATION.md`).
- **Stated goal (from you):** reduce audible clicking in the receiver that coincides with the waterfall updating.
- **Method:** full read of every changed file, diff against upstream, static reasoning about the audio path. Nothing was compiled or run; no hardware. Findings are labelled **Confirmed** (verified by reading the code or filesystem) or **Likely** (inferred, would need hardware or a build to prove). Web sources on the clicking problem could not be opened (network egress blocked), so they are listed as unverified search summaries only.

---

## 1. Verdict in five sentences

The rebuild is a shotgun fix: it defaults the whole panadapter off (`PAN OFF`), which removes every piece of periodic work tied to the waterfall on both the Pi and the Pico. If the clicks come from any of those pieces, the clicks will stop, but the code does not identify which piece was responsible, and the one periodic activity it leaves untouched (the panel's 5 Hz WiFi poll) is itself a candidate. Several of the new features do not work as documented: the rebuilt panel can never display a waterfall at all, `SCAN` returns stale data in the recommended `PAN OFF` mode, `SHUTDOWN` from the panel is a silent no-op on the Pi, and five new buttons sit on top of the F5/F6/F7/WPM/PITCH buttons in CW mode. Three changes in the Pi app that the README does not mention alter transmitter behaviour (key-up before the TX frequency is set, a removed supply-voltage guard, and a changed 15 m low-pass filter selection) and need a deliberate decision. Every bug from the stock panel evaluation carries over unchanged, including the missing `fonts/` directory that stops a clean clone from building.

---

## 2. The clicking problem: what the code says

### 2.1 What happens per waterfall update in the stock V2 system

The panel sends `?\n` every 200 ms (`zbitx_front_panel_v2.ino`, `send_updates`), so the waterfall path runs at 5 Hz. Each cycle touches four places:

| Where | What | Cost (from code) |
|---|---|---|
| Pi audio thread (`sbitx.c` `sound_process`, per 1024-sample block at 96 kHz = every 10.67 ms) | Window multiply + second 2048-point FFT + `spectrum_update` (534 bins of `cabs` and `log10f`) | Small but runs inside the real-time audio callback, 94 times a second |
| Pi GTK thread (`sbitx_gtk.c` `ui_tick`, every 100 ms, 200 ms in FT8) | `draw_spectrum` + `draw_waterfall`: memmove of width x height x 3 bytes, per-pixel colour loop, cairo paint | A few ms on a Pi Zero 2 W; runs at **SCHED_FIFO maximum priority** (`sbitx_gtk.c:5561-5563`) |
| Pi remote thread (`remote.c` `remote_update`) | Builds and sends the 250-byte `WF` line plus every changed field, one `send()` each, TCP_NODELAY | Also **SCHED_FIFO maximum** (`remote.c:127-131`) |
| Pico (`waterfall.cpp`, `screen_gx.cpp`) | `waterfall_update` then a 240 x 96 pixel DMA blit: 69,120 bytes over SPI at 66 MHz | **About 8.4 ms of continuous SPI clocking** per update, plus the WiFi radio burst for the poll/response |

There are four threads at SCHED_FIFO max on a four-core Pi Zero 2 W: the audio thread (`sbitx_sound.c:1016-1020`), the loopback thread (`:1070-1074`), the GTK main thread, and each remote client thread. Only the audio thread has a real-time deadline.

### 2.2 Three candidate mechanisms

**A. Pi-side audio dropout (xrun).** The audio thread has one 10.67 ms period of slack. If the GTK thread's waterfall draw or the remote thread's send burst holds the core the audio thread needs, an ALSA underrun follows and `snd_pcm_recover` (`sbitx_sound.c:835`) produces exactly a click. Linux normally migrates a runnable FIFO thread to an idle core, so on four cores this is not guaranteed to happen, but running the GTK thread at FIFO max is a self-inflicted risk. **Likely contributor; not provable from code.**

**B. Electrical: LCD and WiFi bursts coupling into the receiver.** An 8.4 ms SPI burst at 66 MHz over a ribbon cable at a 5 Hz repetition rate is a textbook source of receiver hash, and the Pico W's WiFi transmit bursts draw supply current spikes at the same 5 Hz rate. The rebuild README's phrase "least LCD/RF activity" suggests the author suspects this too. **Plausible; cannot be evaluated from code at all.**

**C. The V1 mechanism (bit-banged I2C stalling the Pi).** One search result (vu3dxr.in, page could not be opened, so unverified) states that on the original zBitx every waterfall update stalled the system and caused audible hash, and that V2 moved to WiFi to fix it. V2 no longer sends the waterfall over I2C, so this mechanism is gone. Note that the si5351 is still driven by bit-banged I2C from the GTK thread (`i2cbb.c`), so any clicks that track **tuning** rather than the waterfall would come from there.

### 2.3 What the rebuild changes, mechanism by mechanism

| Change | Affects A (Pi CPU) | Affects B (Pico LCD/WiFi) | Notes |
|---|---|---|---|
| `PAN OFF` default: skips spectrum FFT in `sound_process` (`sbitx.c:773-779`) | Yes | No | Confirmed |
| `PAN OFF`: GTK stops drawing spectrum/waterfall (`sbitx_gtk.c:4508-4519`) | Yes | No | Confirmed |
| `PAN OFF` or `WFVIEW WEB`: `remote_get_spectrum` returns empty, no `WF` line to panel (`sbitx_gtk.c:4190-4195`) | Minor | Yes (no LCD blit, smaller packets) | Confirmed |
| Panel frees the 48 KB waterfall buffer when off (`waterfall.cpp`) | No | No | Memory only |
| Panel never makes the `WF` field visible in any mode (see 4.1) | No | Yes: **no LCD waterfall blit ever happens, even with PAN ON** | Confirmed; almost certainly unintended |
| `?` poll every 200 ms unchanged (skipped only while a FREQ send is pending) | No | **No** | The 5 Hz WiFi burst continues |
| GTK/remote threads still SCHED_FIFO max | **No** | No | Unchanged from upstream |
| `DNR` | No | No | Masks noise; does not remove a click source |

**Bottom line:** if the clicks stop with the rebuild, you still will not know why, and if they persist at 5 Hz with `PAN OFF`, the WiFi poll or the power rail is the remaining suspect. The rebuild also cannot be used to test hypothesis B because its panel never blits the waterfall.

### 2.4 Tests that discriminate (in order, cheapest first)

1. **Stock firmware, panel powered off entirely, listen.** No clicks → the panel (LCD, WiFi, or its supply) is the source. Clicks continue → Pi-side.
2. **Stock firmware, `WF OFF` on the panel (stock toggle).** This stops the LCD blit but keeps the Pi FFT, GTK draw, and the 5 Hz poll. No clicks → LCD SPI burst (B). Clicks continue → go to 3.
3. **Check ALSA underruns while clicks occur:** `cat /proc/asound/card*/pcm*p/sub0/status` repeatedly, or build with `DEBUG 1` in `sbitx_sound.c` to print recover events. Underruns that line up with clicks → mechanism A, confirmed. No underruns but clicks → electrical.
4. **Lower the GTK thread priority** (delete `sbitx_gtk.c:5561-5563`) and repeat 3. If underruns vanish, that is the fix and the panadapter can stay on.
5. **Rebuild firmware, `PAN ON` + `WFVIEW ALL` set from the web UI.** The Pi does all its work and sends `WF`; the Pico parses it but never blits (4.1). Clicks → Pi or network. No clicks → the LCD blit was the source.
6. **Record with `REC` while clicks occur** and inspect the file. Clicks in the recording rule out the speaker amplifier and codec output stage; they do not distinguish RF pickup from xruns.

### 2.5 Targeted fixes, depending on the answer

- **If A:** run only the audio thread as SCHED_FIFO; move the display FFT out of `sound_process` (copy the block, do the FFT in the GTK or remote thread); pin the audio thread to one core and keep the others off it (`pthread_setaffinity_np`); consider 3 periods per buffer. All of these keep the waterfall.
- **If B (LCD):** drop `SPI_FREQUENCY` from 66 MHz to 20-30 MHz in `TFT_setup.h` / `tft_ili9488.cpp`; blit the waterfall in 12-row chunks spread across the 200 ms window instead of one 8.4 ms burst; add series resistors or a ferrite on the LCD ribbon (hardware).
- **If B (WiFi):** lengthen the poll to 500 ms or move to a push model where the Pi sends only on change; add bulk capacitance at the Pico W.

---

## 3. Pi application findings (`zbitx2-rebuild`)

**Attribution caveat.** Neither `afarhan/zbitx` branch on GitHub contains the WiFi `remote.c` thread server or `center_bin`; the public upstream still drives the panel over I2C. The base this rebuild started from is therefore not public, and "differs from upstream" below does not mean "changed by the rebuild". The README's feature list is the only record of intent. The repo has one squashed commit, so history cannot answer it either.

### 3.1 High

**3.1.1 DNR use-after-free across threads.** *(Confirmed, detailed in the prior message.)* `dnr_set_enabled(0)` (`sbitx.c:280-291`) frees `dnr_noise`/`dnr_gain` from whichever thread ran `sdr_request` while the audio thread may be inside the `dnr_process` loop (`:314-351`). No mutex exists in `sbitx.c`. Fix: allocate once at startup, never free.

**3.1.2 `SCAN` returns stale data when `PAN` is off.** *(Confirmed.)* `spectrum_plot[]` is written only in `spectrum_update` (`sbitx.c:378`), which runs only inside `if (panadapter_enabled)` (`:773`). `remote_get_bandscope` (`sbitx_gtk.c:4243-4288`) reads `spectrum_plot`. The README recommends `PAN OFF` plus `SCOPE`/`SCAN` for a quick band look; in that mode `SCAN` sends whatever spectrum was last computed while `PAN` was on, or zeros. The headline low-CPU workflow does not work.

**3.1.3 `tx_on` keys the transmitter before setting the TX frequency.** *(Confirmed vs upstream main.)* Upstream order was: set `in_tx`, `set_operating_freq`, then `sdr_request("tx=on")`. The rebuild (`sbitx_gtk.c` `tx_on`) issues `tx=on` first, then sets `in_tx` and retunes. With SPLIT or RIT this transmits briefly on the receive frequency. The README does not mention this change.

**3.1.4 Supply-voltage TX guard removed.** *(Confirmed vs upstream main.)* Upstream `tx_on` refused to transmit on hardware version 4 when `#batt` exceeded 900 (9.00 V) and printed "Reduce the power supply voltage to transmit". The rebuild deletes that block. `data/hw_settings.zbitx_v2` sets `hw=4`. If the guard protected the PA, this is a hardware-risk change; if it was a nuisance false positive, it should be documented. I cannot tell which from the code.

**3.1.5 15 m low-pass filter selection changed for hw 4.** *(Confirmed vs upstream main.)* `sbitx.c:444-445` comments out `else if (frequency < 21500000 && sbitx_version >= 4) lpf = LPF_B;`. On `hw=4`, 21.0-21.45 MHz now selects `LPF_A` instead of `LPF_B`. Whether that is correct depends on the V2 filter board cutoffs; if wrong, second-harmonic suppression on 15 m suffers. Not mentioned in the README.

### 3.2 Medium

- **`WFVIEW` defaults to `WEB`, and the panel has no way to change it.** `remote_get_spectrum` returns nothing unless `waterfall_local_enabled()` (PAN on **and** WFVIEW ALL). The panel's `WFVIEW` field is placed off-screen at (20000, 20000) in `fields_list.h`, so from the panel `PAN ON` alone never produces a waterfall; the operator must set WFVIEW from the GTK or web UI. *(Confirmed.)*
- **`SHUTDOWN` from the panel is a no-op.** The panel sends `SHUTDOWN`. On the Pi, `cmd_exec` (`sbitx_gtk.c:5220`) resolves unknown commands by field **label** (`:5413`), and the shutdown field's label is `SHDN` (`:534`); the only `"SHUTDOWN"` handler is inside `do_control_action` (`:4897`), which `cmd_exec` reaches only through a matched field. Result: nothing happens, silently. Sending `SHDN` instead would reach `do_control_action("SHDN")`, which opens a **modal GTK confirmation dialog** on a radio that has no screen or mouse. *(Confirmed.)* Fix: add `else if (!strcmp(exec, "SHUTDOWN")) clean_system_shutdown();` to `cmd_exec`, since the panel already confirmed.
- **TX bandscope reads stale modulation data.** `sdr_modulation_update` is now gated on `panadapter_enabled` (`sbitx.c:1061`), but `remote_get_bandscope` in TX reads `mod_display` regardless. *(Confirmed.)*
- **FREQ coalescing can reorder commands.** `remote_execute` parks `FREQ` in a one-slot mutex-protected pending buffer and queues everything else; `ui_tick` applies the pending FREQ before draining the queue. A `VFO B` received before a `FREQ` in the same 1 ms window is applied after it. Rare; low impact. The coalescing itself is a sound design. *(Confirmed.)*
- **Remote client threads run at SCHED_FIFO max** (`remote.c:127-131`) to do blocking `recv` and socket sends. They should be SCHED_OTHER. *(Confirmed.)*
- **`remote.c:172` compares the whole receive buffer to `"OPEN "`** instead of the current token `t`; works only when `OPEN ` arrives alone in its TCP segment. *(Confirmed.)*
- **`i2cbb.c` "mutex" is a busy-wait** of up to 100 x 10 ms = 1 s inside the GTK thread when two callers collide. *(Confirmed.)*
- **`get_field_timestamped` bound check uses `>` where `>=` is needed** (`sbitx_gtk.c:1003`); harmless only because the layout array has a terminator. *(Confirmed.)*

### 3.3 Low

- README says the matching panel firmware "must include the same command set for PAN, DNR, DNRLVL, SCOPE, SCAN, BS, and SHUTDOWN"; SHUTDOWN and BS-with-PAN-OFF do not work end to end (above).
- `Makefile` and `build` duplicate each other; `CLAUDE.md` documents `./build`, README documents `make`.
- Debug `printf` added in the RIT path (`sbitx_gtk.c:3297`).
- `hostname`, `hosts`, `sample.store`, a 1.1 MB compiled `sbitx` binary, an 8 MB STL, and several PDFs are committed. The binary will go stale.
- Upstream drift the README does not list (attribution unknown, mostly beneficial): NTP moved to a background thread, logbook NULL guards from SQLite, `record=off` NULL check, FT8 logs the QSO on RR73 instead of RRR, RIT/SPLIT now retune the synthesizer immediately, `sdr_request` NULL-pointer check fix.

### 3.4 What is good

- `PAN OFF` gating is implemented consistently across the audio thread, GTK, web, and remote paths.
- The FREQ coalescing (latest wins, applied on the 1 ms UI tick) is the right shape for encoder tuning.
- DNR and PAN are deliberately excluded from saved settings so the radio always boots in the quiet state.
- The clean-shutdown sequence (stop TX, stop recording, save, `sync`, `systemctl poweroff` via fork/exec, checked return) is correct where it is reachable.

---

## 4. Front-panel findings (`zbitx2-front-rebuild`)

Only five source files differ from the stock panel: `zbitx_front_panel_v2.ino`, `fields.ino`, `fields_list.h`, `waterfall.cpp`, `zbitx.h`, plus new `bandscope.cpp`.

### 4.1 High

**4.1.1 The waterfall can never be displayed.** *(Confirmed.)* `field_set_panel` (`fields.ino:144-150`) replaced `WF` with `PAN` in the FT8 list and with `PAN/SCOPE/SCAN/DNR/DNRLVL/.../BS` in the CW and voice lists. `WF` is not in the first 20 always-on fields either. `field_draw_all` only draws visible fields, so `waterfall_draw` never runs. `PAN ON` on the panel allocates 48 KB, tells the Pi to do the FFT work, and (with WFVIEW ALL) receives and parses `WF` lines into a buffer nobody blits. The README's "PAN ON/OFF front-panel waterfall/panadapter control" is not true as shipped.

**4.1.2 Button collisions in CW and FT8 modes.** *(Confirmed by coordinates in `fields_list.h`.)* In CW/CWR the panel shows `F1-F7`, `PITCH`, `WPM` **and** the new row. Same 48 x 48 cells:

| Cell (x, y=272) | Existing | New |
|---|---|---|
| 240 | F5 | DNR |
| 288 | F6 | DNRLVL |
| 336 | F7 | PAN |
| 382/384 | WPM | SCOPE |
| 432 | PITCH | SCAN |

`field_at` returns the first visible match in list order, and the F-keys, WPM, and PITCH come first, so DNR, DNRLVL, PAN, SCOPE, and SCAN are unreachable by touch in CW mode and the cells draw twice. In FT8, `PAN` (336) sits on `TX1ST` (336). Voice mode has no collisions.

**4.1.3 Cross-core use-after-free on the malloc'd waterfall buffer.** *(Confirmed paths; race window is small.)* The stock static array became a pointer. Core 1 (touch) toggles `PAN` in `field_select` and calls `waterfall_set_enabled(false)` → `free` (`fields.ino:300`, `waterfall.cpp:35-40`). Core 0 (network) runs `field_set("WF")` → `waterfall_update`, which `memset`s and `memmove`s the same buffer (`fields.ino:218-244`, `waterfall.cpp:150-188`). No lock. A PAN OFF tap during a WF update frees memory core 0 is writing.

**4.1.4 All four High bugs from the stock evaluation are still present**, at shifted line numbers: unchecked `strcpy` into `f->value` from the radio (`fields.ino:215`, `:252`, `:259`), NULL dereference on `f_selected->label` (`fields.ino:352`), TCP read off-by-one (`zbitx_front_panel_v2.ino:573`), and the core-1 watchdog defeated by core 0 (`:581`). The EEPROM double-increment (`storage.cpp:53`) and the missing `fonts/` directory (`tft_ili9488.cpp:80-82`) are also unchanged, so the README's `arduino-cli compile` line fails on a clean clone.

### 4.2 Medium

- **`SHUTDOWN` confirmation dialog posts a command the Pi ignores** (see 3.2). The two-step dialog in `field_select` (`fields.ino:322-330`) is well built; the Pi side is the gap.
- **`WFVIEW` is a hidden field** (20000, 20000) so the operator cannot select ALL from the panel; combined with 4.1.1 it does not matter yet, but it will once the waterfall is visible again.
- **Tuning rate-limit logic is sound but slightly inconsistent:** `front_panel_tuning_active()` compares `unsigned long now` with `unsigned int last_wheel_moved`; fine for 49 days of uptime. The FREQ-first send path skips the `?` poll while a frequency is pending, which is a nice side effect for hypothesis B during tuning.
- **`bandscope_update` clamps to 63 levels but the Pi encodes 0-63 into 32-95**; consistent. The bandscope draws 80 one-pixel lines with `screen_draw_line`, each a separate SPI transaction. Fine for a one-shot.

### 4.3 Good

- `waterfall_line` bound fixed from `> 240` to `>= 240`, and the scroll write now uses `(f->w - 1) - i`, fixing the one-past-the-row writes I flagged in the stock evaluation.
- The bandscope is a genuinely low-churn design: one 83-byte line, one draw, no scrolling.
- README is a large improvement over the stock one-liner.

---

## 5. Protocol consistency between the two repos

| Command | Panel sends | Pi handles | Works? |
|---|---|---|---|
| `PAN ON/OFF` | `field_select` toggle → `PAN ON` | `cmd_exec` → label `PAN` → `set_field("#pan")` → `do_control_action("PAN ON")` (`:4990`) | Yes (Confirmed) |
| `DNR ON/OFF`, `DNRLVL n` | selection/number fields | label match → `set_field` → generic `sdr_request` (`:5116-5119`) → `dnr=ON` / `dnrlvl=n` | Likely yes (generic path read, exact string not traced) |
| `SCOPE ON/OFF` | selection | `do_control_action("SCOPE ON")` (`:4984`) | Yes (Confirmed) |
| `SCAN` | button → `SCAN ` | label `SCAN` → button → `do_control_action` twice (`SCAN`, then `SCAN `) | Yes, requests twice, harmless; **data stale when PAN OFF** |
| `BS <80 chars>` | receives | `remote_get_bandscope` only when `bandscope_pending` | Yes |
| `WF <250 chars>` | receives, never draws | sent only when PAN ON and WFVIEW ALL | **Never visible** (4.1.1) |
| `SHUTDOWN` | after two confirmations | no label `SHUTDOWN`; handler unreachable | **No** (3.2) |
| `FREQ n` | coalesced, sent first | `is_freq_command` → pending slot → `ui_tick` | Yes (Confirmed) |

---

## 6. Recommended order of work

1. Decide the clicking question with tests 1-3 in section 2.4 before changing anything else. Two of the three are done in five minutes with stock firmware.
2. If test 3 shows underruns, remove the SCHED_FIFO promotion of the GTK thread (`sbitx_gtk.c:5561-5563`) and of the remote threads (`remote.c:127-131`), then retest. This alone may let the panadapter stay on.
3. Make the DNR buffers permanent (3.1.1).
4. Add `WF` back to the panel mode lists and move the five new buttons off the F-key row, or put them behind a dialog like MENU/SET (4.1.1, 4.1.2).
5. Add a `SHUTDOWN` case to `cmd_exec` (3.2).
6. Make `SCAN` compute a spectrum on demand: run the display FFT once for the next audio block when `bandscope_pending` is set, regardless of `PAN` (3.1.2).
7. Restore or document the three transmitter-side changes (3.1.3-3.1.5). `tx_on` order is a two-line revert.
8. Fix the inherited panel bugs listed in 4.1.4 and commit `fonts/`.

---

## 7. Not verified

- Which of mechanisms A or B produces the clicks. Code review cannot decide it; section 2.4 can.
- The Pi model (Zero 2 W) is taken from a comment in `setup-ap.sh`; the 96 kHz sample rate from comments in `sbitx.c`. Neither was confirmed from a running system.
- The V1 stall-and-hash description comes from a search summary of vu3dxr.in that I could not open; treat it as hearsay. A second hit, a Facebook post titled "I found that all the noise is caused by the front panel", also could not be opened.
- Whether `LPF_A` is correct for 21 MHz on V2 hardware (3.1.5) and whether the 9 V guard (3.1.4) protected anything.
- The exact string `set_field` passes to `sdr_request` for the DNR fields.

## Sources (search summaries only; pages blocked from this environment)

- https://vu3dxr.in/the-zbitx-v2-has-arrived-efficiency-built-in-efhw/
- https://vu3dxr.in/real-world-feedback-navigating-zbitx-v2-issues-upgrades/
- https://www.hfsignals.com/index.php/zbitx-v2/
- https://www.facebook.com/groups/zbitx/posts/1238376614622281/
