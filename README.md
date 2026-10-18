# Emulator Console v2

A handheld Game Boy (DMG) emulator console built around an **ESP32-S3**, designed from scratch in KiCad. The target game is Pokémon Red/Blue.

v2 is a ground-up redesign of the first board (v1, "Rev A"). v1 never booted, and the redesign fixes everything that went wrong with it. Every part on v2 came from parts already on hand. Nothing new was bought for this revision.

---

## At a glance

| | |
|---|---|
| MCU | ESP32-S3-WROOM-1U-N8R8 (dual-core 240 MHz, 8 MB flash, 8 MB octal PSRAM). WiFi unused |
| Display | 2.4" 320×240 IPS, ST7789, SPI at 80 MHz, landscape |
| Storage | microSD, 1-bit SDMMC, with card detect |
| Audio | MAX98357A I²S class-D amp → 8 Ω 1 W speaker |
| Controls | D-pad, A, B, Select, Start, L, R (10 buttons), plus RESET and BOOT |
| Power | 1-cell Li-ion with built-in protection, USB-C charging at 500 mA, play-while-charging |
| Board | 70 × 95 mm, 4-layer, 1.6 mm, JLC04161H-7628 stackup |
| Debug | Native USB (Serial/JTAG), UART header, labelled test points, rail-OK LEDs |

---

## How v2 differs from v1

### Why there's a v2

The v1 board never powered up properly:

- It **never enumerated over USB**, and plugging it into a laptop **crashed the laptop**.
- The buck-boost regulator **ran hot on battery alone**.
- With the board unpowered, **+3V3 measured 0.4 Ω to GND**. That's a dead short on the main rail.

There was no way to isolate the fault. v1 had no test points, no debug UART, and the only regulator was a fine-pitch VSON package. Inspection also found heavy uncleaned flux, loose solder balls and a solder bridge on the module.

Debugging it also exposed several design-level problems: no audio on battery, no power path, and a write-only display bus. So rather than patch v1, v2 started from a blank schematic.

### Change summary

| Area | v1 (Rev A) | v2 | Why |
|---|---|---|---|
| **3.3 V regulation** | TPS63020 buck-boost, VSON-14 | **Two AP2112K-3.3 LDOs**: one for the MCU, one for peripherals | A buck-boost only recovers the last ~5% of a Li-ion cell's capacity. Two LDOs are simpler, have no switching layout to get wrong, are easy to hand-solder and rework, and each one stays under ~150 mA |
| **Power path** | Battery → switch → system | **USB-priority load sharing** (SS24 + AO3401) | The system runs from USB while the cell charges undisturbed. **The board runs from USB with no battery fitted** |
| **Power switch** | Slide switch carried the full battery current | **Slide switch drives both LDO enable pins** | The switch (rated ~0.5 A) now carries microamps |
| **Charge current** | 1 A | **500 mA** (2 × 1.2 kΩ PROG) | Stays inside a default USB 2.0 port budget, and the TP4056 runs cooler |
| **Amp supply** | `+5V_USB`: **no audio on battery** | `VSYS` | Audio works on battery |
| **Amp control** | Always on, 9 dB gain, (L+R)/2 mode via 1 MΩ | **Enabled by GPIO46** through 1 kΩ (left-channel mode), **6 dB gain** | No pop at boot and no idle current. Lower gain protects the 1 W speaker. Firmware sends mono audio |
| **Display bus** | SPI on arbitrary pins, write-only | **SPI2 dedicated pins** (SCK IO12, MOSI IO11) + **FMARK** | Reliable 80 MHz, which is the emulator's frame-time bottleneck. FMARK enables tear-free updates |
| **Display mounting** | Flat footprint | **Landscape, flex wrapped around the board edge** to pads on the back (custom footprint) | The screen is centred on the front, and all other parts live on the back |
| **Backlight** | Anode straight to 3.3 V, no current limit | **Series resistor R23** + low-side FET with PWM | The LED forward voltage is as low as 2.8 V. Without a resistor, current depends on the panel batch |
| **microSD** | 4-bit SDMMC, no card detect | **1-bit SDMMC + card detect** | 1-bit mode loads a 1 MB ROM in under half a second and frees 3 GPIOs |
| **Buttons** | 8 | **10** (adds L/R shoulders on the back) | Menu and save-state shortcuts without button chords |
| **Battery sense** | None | **÷2 divider (1 MΩ/1 MΩ + 100 nF) → IO16** | Battery gauge and low-battery shutdown |
| **Charge status** | CHRG/STDBY tied to GND | **CHRG → IO9** with a 10 kΩ pull-up to 3.3 V | Firmware can show a charging icon |
| **Speaker** | JST PH header (the speaker plug is actually 1.25 mm pitch) | **Solder pads with strain relief** | The original connector never fitted |
| **Debug** | USB only | **UART header, labelled test points, rail-OK LEDs** | There's always a way in, and faults can be found by probing |
| **PCB** | Rectangular, Select/Start on the left edge, no mounting holes | **4-layer, 70 × 95 mm**, centred controls, 4 × M2 holes | Solid ground plane, better ergonomics, ready for an enclosure |

---

## Hardware

### Power

```
USB-C ── USBLC6 ESD ── D+/D− ─────────────────────────── ESP32-S3 (IO19/IO20)
  │
+5V_VBUS ──┬──► TP4056 (500 mA) ──► VBAT ──► AO3401 (on when no USB) ──┐
           │                                                           │
           └──► SS24 ───────────────────────────────────────────────────┴──► VSYS (3.0–4.6 V)
                                                                              │
                         ┌──────────────────────────┬─────────────────────────┤
                AP2112K → +3V3_MCU          AP2112K → +3V3_PERIPHERAL     MAX98357A
               (EN ← slide switch)          (EN ← slide switch)
                   ESP32-S3                   display, backlight, microSD
```

- **On USB:** the SS24 feeds VSYS, the AO3401 is held off (its gate is on VBUS), and the TP4056 charges the cell independently.
- **On battery:** a 10 kΩ resistor pulls the AO3401 gate low, so the cell feeds VSYS through about 50 mΩ.
- **Switch off:** both LDOs shut down. The amp sits in shutdown because GPIO46 is low. Total drain is a few µA.
- **Charging works with the switch off.**
- **Low-battery protection:** the cell's own protection circuit plus a firmware cutoff. The LDOs drop out gracefully, so the board stays usable down to about 3.0 V.

### Boot and strap pins

| Pin | How it's handled |
|---|---|
| EN | 10 kΩ pull-up + 1 µF (~10 ms delay), RESET button, test point |
| IO0 | 10 kΩ pull-up, BOOT button, test point |
| IO45 | 10 kΩ pull-down, which selects 3.3 V VDD_SPI for the N8R8. It also drives the backlight gate, so the screen stays dark until firmware lights it |
| IO46 | Held low at reset by the amp's internal 100 kΩ pull-down through the 1 kΩ |
| IO3 | Test point only; its strap function is inactive unless an eFuse is burned |
| IO35–37 | Left unconnected (used inside the module by the octal PSRAM) |

**Flashing:** over USB-C. If USB doesn't enumerate, connect a USB-UART adapter to the header (`3V3 GND TX RX`), hold **BOOT**, tap **RESET**.

### GPIO map

| GPIO | Function | | GPIO | Function |
|---|---|---|---|---|
| 0 | BOOT (strap) | | 21 | TFT FMARK |
| 1 | R shoulder | | 38 | B |
| 2 | Select | | 39 | A |
| 3 | Test point | | 40 | I²S DIN |
| 4 | Up | | 41 | I²S BCLK |
| 5 | SD card detect | | 42 | I²S LRCLK |
| 6 | SD DAT0 | | 43 | UART TX (header) |
| 7 | SD CLK | | 44 | UART RX (header) |
| 8 | Right | | 45 | Backlight PWM (strap, pulled down) |
| 9 | CHRG (charging = low) | | 46 | Amp enable (SD_MODE) |
| 10 | TFT DC | | 47 | Start |
| 11 | TFT MOSI (SPI2 dedicated pin) | | 48 | L shoulder |
| 12 | TFT SCK (SPI2 dedicated pin) | | 19 / 20 | USB D− / D+ |
| 13 | TFT CS | | 35–37 | Octal PSRAM (do not use) |
| 14 | TFT RST | | | |
| 15 | SD CMD | | | |
| 16 | VBAT sense (ADC2; WiFi stays off) | | | |
| 17 | Down | | | |
| 18 | Left | | | |

Buttons short to GND and use the internal pull-ups, with debouncing in firmware.

### PCB

- **Stackup:** F.Cu (front, mostly buttons) / In1 power areas (VBUS, VSYS, +3V3_MCU, +3V3_PERIPHERAL) / In2 solid GND / B.Cu (main routing).
  - The back layer, where almost everything is routed, sits right over the solid ground plane.
- **Front:** display and the 8 face buttons only.
  - Nothing through-hole sits behind the glass, so the front stays flat.
- **Back:** everything else. The module sits behind the screen, with power at the bottom by USB-C and audio in the bottom right.
- **USB:** D+/D− are length-matched (52.1 mm each), 0.2 mm wide with a 0.25 mm gap. That calculates to about 105–113 Ω differential. It's above the 90 Ω target, but that doesn't matter at full-speed USB (12 Mbps).
- **Other layout details:**
  - GND stitching vias around the edge and through open areas, plus teardrops.
  - Thermal vias under the TP4056, the amp and the module's ground pad.
  - Vias are covered by solder mask (tented).

### Display flex

The panel's flex can't twist, so it wraps around the board's **side edge** to solder pads on the back. A custom footprint places the glass centred on its anchor, with the 16 pads on B.Cu at the calculated fold position. The pads are 5 mm long, which absorbs tolerance in the flex height.

**Important:** don't use KiCad's *Flip* on this footprint. A fold around the edge doesn't mirror the pin order, but Flip would.

---

## Bring-up order

Each stage adds one subsystem, so a fault points at whatever was just added.

0. **Harvest parts from v1.** The module, LCD, amp, slide switch and speaker are quantity 1 and come off the old board. Measure the bare module's 3V3 pin to GND first: a high reading means it's good; about 0.4 Ω means it's dead.
1. **Power only.** Check every rail for shorts before applying power. Then:
   - **USB with no battery:** VSYS ≈ 4.6 V, both rails at 3.3 V, both rail LEDs lit.
   - **Switch off:** the rails go to 0.
   - **Battery connected:** charging works.
   - **Unplug USB:** VSYS follows the battery.
2. **MCU.** It should enumerate as *USB JTAG/serial debug unit*. Run the rail, button and battery tests.
3. **Display.** Fill the screen, then step SPI from 40 to 80 MHz.
4. **microSD.** Mount the card; test card detect.
5. **Audio.** Test **on battery, with USB unplugged**. That was v1's main defect.

---

## Firmware notes

ESP-IDF v5.5. Key `sdkconfig` settings:

```
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y
```

- **Display:** SPI2 host, ST7789 with INVON (0x21). Rotate the image for landscape. Sync frame transfers to FMARK.
- **SD:** SDMMC with `slot_config.width = 1` (CLK 7 / CMD 15 / D0 6), plus card detect on IO5.
- **Audio:**
  - Drive IO46 high before starting I²S and low after stopping it.
  - Use `I2S_SLOT_MODE_MONO`; the amp runs in left-channel mode.
  - **Cap the volume.** On USB, VSYS is 4.6 V, which can push more than 1 W into the speaker.
- **Battery:** read IO16, ×2 for the cell voltage. Sample no faster than every few hundred ms. Save and power down below about 3.4 V.
- **Planned:** USB mass-storage mode, so ROMs can be copied over USB-C without opening the case. The microSD slot sits inside the enclosure.

---

## Known limitations

- **WiFi is not supported.** Transmit peaks (350–500 mA) would overload the MCU's LDO.
- **The microSD card is only reachable with the case open** until USB mass-storage mode is implemented.
- **Several parts are quantity 1**, harvested from v1: the ESP32-S3 module, LCD, MAX98357A, slide switch and speaker. A second fully populated board would need new ones.
