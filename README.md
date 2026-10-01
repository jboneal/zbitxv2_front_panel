# zBitx V2 Front Panel (Pico) — stock firmware with a wired USB link

The TFT driver files `tft_ili9488.*` are extracted from the TFT_eSPI library
and tailored for the zBitx front-panel display.

## Wired Link To The Radio (replaces WiFi)

The Pico W's WiFi transmissions were traced to audible clicks in the receiver:
the clicks stop when the panel's WiFi stops, with the display still running.
This branch therefore carries the same line protocol over the panel's USB
(CAT) port and never starts the WiFi radio. Everything else — fields,
waterfall, FT8, logbook, the 200 ms `?` poll — is the stock code.

Select the link in `zbitx.h`:

```c
#define PANEL_LINK PANEL_LINK_USB    // default on this branch
// PANEL_LINK_WIFI  -> original behaviour
// PANEL_LINK_UART  -> GP16/GP17; not usable on the zBitx V2 main board, see below
```

### USB cable (no wiring changes)

Connect the panel's CAT/USB port (the one used for flashing) to the Pi's USB
host port on the back of the radio with a short shielded USB cable. The Pico
enumerates on the Pi as a CDC serial device. The panel board feeds the Pico's
VSYS through a diode from its own regulator, so being powered by both the
radio and the Pi's USB is the Pico datasheet's diode-OR arrangement; measure
VSYS once to confirm the V2 panel kept that diode (expect about 4.4 V with
the cable out, rising to about 4.7 V with it in).

**Build target: `rp2040:rp2040:rpipico` (plain Pico), on purpose.** The board
is a Pico W, but building for the non-W variant leaves the CYW43 radio chip
unpowered. Nothing in this firmware touches the W's LED or radio. Use
`rpipicow` only for `PANEL_LINK_WIFI`.

Debug output moves to the UART on GP16/17 (unconnected on the stock panel).
Every `Serial.print` in the firmware now goes through the shared `Debug`
stream, so nothing but protocol reaches the Pi. BOOTSEL flashing is
unchanged: hold the knob at power-on with the cable moved to a PC.

### Pi side

Either of these, with no change to how the panel behaves:

1. **socat bridge, radio software untouched:**
   `socat /dev/zbitx-panel,raw,echo=0 TCP:127.0.0.1:8081` as a systemd
   service with `Restart=always`.
2. **In-process serial transport** in `remote.c` (branch
   `claude/serial-panel-link` of `jboneal/zbitx2-rebuild`):
   `panel_serial=/dev/zbitx-panel` in `~/sbitx/data/hw_settings.ini`.

Give the port a fixed name and keep ModemManager off it:

```
# /etc/udev/rules.d/99-zbitx-panel.rules
SUBSYSTEM=="tty", ATTRS{idVendor}=="2e8a", ENV{ID_MM_DEVICE_IGNORE}="1", SYMLINK+="zbitx-panel"
```

### Why not the GPIO UART

The Pi's hardware UART is BCM14/BCM15 (header pins 8/10). On the zBitx V2
main board BCM15 is `RX_LINE`, the T/R switching output (`sbitx.c`
`#define RX_LINE 16`, wiringPi numbering). Enabling `/dev/serial0` breaks
transmit/receive switching. The bit-banged I2C pair in the existing 10-wire
panel cable carries the si5351 and the OLED, so it cannot be repurposed
either. `PANEL_LINK_UART` stays in the code for other boards.

## Build

Arduino IDE or `arduino-cli` with the Earle Philhower `arduino-pico` core
(2.x or newer; `Serial1.setFIFOSize` is used in UART mode). The sketch folder
must be named `zbitx_front_panel_v2`.

The `fonts/` folder the driver includes (`glcdfont.c`, `Font16.c`,
`Font32rle.c`) is not committed. Copy them from Bodmer/TFT_eSPI `Fonts/`, and
in `glcdfont.c` remove `static` from the `font[]` definition so it matches the
`extern` declaration in `tft_ili9488.h`.

```sh
arduino-cli compile --fqbn rp2040:rp2040:rpipico .
```

Not built here: this branch was written and preprocessor-checked in all three
link modes but not compiled with the Arduino toolchain.

## Flash

1. Connect the panel's USB to a PC.
2. Power on while holding the tuning knob; the panel mounts as `RPI-RP2`.
3. Copy the `.uf2` onto it. The committed `zbitx_front_panel_v2.ino.uf2` is
   the stock WiFi build, kept for rollback.

See `CODE_EVALUATION.md` and `REBUILD_EVALUATION.md` for the code reviews
that led here.
