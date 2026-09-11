# Plan: Converting the Alt-PCB into a picoTracker "Advance" Equivalent

*Drafted 2026-09-10, based on the alt-PCB `revised-pcb` branch, the public
[xiphonics/picoTracker](https://github.com/xiphonics/picoTracker) firmware
(local clone @ `98a96af`, 2026-08-14), and public Advance product info.*

> **Decisions locked 2026-09-10:**
> - MCU module: **Pimoroni Pico LiPo 2 (non-XL, PIM775)** — drop-in for the
>   existing Pico footprint. Pin budget closes by converting the 9 keys to a
>   **3×3 diode matrix** (see §4). (The XL was considered for its extra
>   GPIO but rejected to keep U1 a drop-in swap.)
> - Display: **keep the existing ILI9341 3.2" 320×240** — avoids driver,
>   font, and mechanical rework (an ST7796S 4.0" upgrade was evaluated and
>   parked; see §6).

## 1. Reality check: what the Advance is, and what is (not) public

The picoTracker Advance (launched Nov 2025) is a **commercial, closed-hardware
product**. What's publicly known:

| Spec | Advance |
| --- | --- |
| Display | 4" HiDPI **720×720** |
| Processor | "Powerful ARM processor" — exact MCU never published |
| Sample memory | **48 MB** |
| Audio out | Headphone/line out + internal speaker |
| Audio in | **Line input + internal microphone** (sampling/recording) |
| MIDI | TRS **and USB** MIDI in/out |
| Channels | 8 stereo channels |
| Storage | microSD |
| Power | USB-C charging, ~6 h battery, accurate battery metering |
| Size | 143×78×15 mm, 233 g |

Crucially, from examining the firmware repo:

- The public repo contains **only the RP2040 board adapter**
  (`sources/Adapters/picoTracker/`). There is no `advance` adapter on any
  public branch — the Advance's board-support code is private.
- The **application layer is shared and public** (BSD-3-Clause): recording
  (`RecordStreamer`, `record.h` with `LineIn`/`Mic`/`USBIn` sources — dummy
  stubs on pico), 8-channel playback, battery metering hooks, USB MIDI, etc.
  The hooks for Advance features exist; the drivers don't.
- Supported displays are **ILI9341 / ST7789, 320×240 character-cell UI** only.
- Only `PICO_PLATFORM=rp2040` build presets exist today (the bundled FreeRTOS
  does ship RP2350 ports, so an RP2350 target is plausible).

**Conclusion:** a literal clone (720×720 screen, 48 MB, unknown MCU) is a
from-scratch product design plus a firmware port with no reference code — not
a "conversion." What *is* achievable as a conversion is **functional parity**:
sampling/line-in/mic, headphone out, big sample memory, 8 solid channels,
USB+TRS MIDI, good battery metering — on the open firmware. Call it
**"Advance-lite."** That is what this plan targets.

## 2. Gap analysis: alt-PCB today vs Advance

| Feature | Alt-PCB today | Advance | Gap |
| --- | --- | --- | --- |
| MCU | RP2040 (Pimoroni Pico LiPo 16MB) | unknown ARM | ⚠ upgrade |
| Sample RAM | 264 KB SRAM (+ `LOAD_IN_FLASH`) | 48 MB | ⚠ **biggest gap** |
| Display | 3.2" ILI9341 320×240 SPI | 4" 720×720 | kept as-is by decision (upgrades parked, §6) |
| Audio out | PCM5102 I2S DAC (line-level) → MAX98306 amp → speakers | HP/line out + speaker | ~ partial (no HP driver / jack) |
| Audio in | **none** (PCM5102 is DAC-only) | line-in + mic | ⚠ needs full codec |
| MIDI TRS | in/out, 6N138 opto, A/B jumpers | in/out | ✅ done |
| USB MIDI | supported by firmware via USB-C | in/out | ✅ done |
| SD | SDIO on GP2–7 (J8) | microSD | ✅ done |
| Keys | 9 keys, same layout as Advance keymap | compatible per DEV.md | ✅ done |
| Battery | LiPo + charger on module, GP29 voltage divider | accurate metering | ~ add fuel gauge |
| Speaker | 2× micro speaker + kill switch | 1 speaker | ✅ done |

## 3. Target spec ("Advance-lite")

- **MCU module (decided): Pimoroni Pico LiPo 2 (PIM775)** — RP2350B, 16 MB
  flash, 8 MB PSRAM, USB-C, MCP73831 charger + XB6096 protection, Qw/ST I2C
  connector, **standard Pico footprint → U1 is a drop-in swap**. Board
  housekeeping lives on non-exposed high GPIOs (BOOT GP45, VBAT sense GP43,
  PSRAM CS internal), so all 26 exposed GPIO (GP0–22, GP26–28) are free for
  the design. Battery voltage remains readable with zero external pins via
  GP43. Trade-off accepted: the exposed pin budget closes exactly and only
  via the key-matrix conversion (§4) — no headroom remains.
- **Display (decided): keep the ILI9341 3.2" 320×240** — fully supported by
  stock firmware (`USE_LCD=LCD_ILI9341`), no driver/font work, no board
  outline or back-panel changes. Larger-screen options stay parked in §6.
- **Full audio codec** replacing the GY-PCM5102 module: stereo DAC + stereo
  ADC + mic input + headphone driver. Candidates:
  - **TLV320AIC3204/3254** (TI): ADC+DAC+PGA mic in+HP driver in one chip, I2C
    control — closest to one-chip Advance-like audio. SMD only.
  - **ES8388 module** (cheap AliExpress/LCSC): ADC+DAC+HP amp, I2C — good
    module-style fit matching this board's THT/module philosophy.
  - **PCM3060** (ADC+DAC, no HP amp/mic bias — pair with MAX9724/TPA6132 HP
    amp and a MEMS mic preamp) — simplest I2S, more parts.
- **Headphone/line-out 3.5 mm jack** with switch contact → mute speaker amp
  (`SD` pin on MAX98306) when plugged.
- **Analog MEMS mic** (e.g. SPW2430 module or ICS-40180) into codec mic input,
  port hole in back panel.
- **I2C fuel gauge** (MAX17048) for accurate battery %, on the same I2C bus as
  the codec (or the Pico LiPo 2's Qw/ST connector).
- Keep: SDIO SD, TRS MIDI circuit, 9-key layout, MAX98306 + speakers, kill
  switch, sandwich-PCB enclosure concept.

## 4. Pin budget (closes exactly — the key matrix is what makes it work)

The Pico LiPo 2 exposes the standard 26 Pico GPIO (GP0–22, GP26–28), and
today's map consumes **all 26**: MIDI GP0/1 · SDIO GP2–7 · keys GP8–16 ·
I2S out GP17–19 · display CS/DC/RST GP20–22 · display SCK/MOSI/MISO
GP26–28. (Old GP23 backlight-PWM and GP29 VBATT entries are module-internal
concerns; on the LiPo 2 battery sense is internal GP43.)

**Recovered pins (4):**

| Change | Pins freed |
| --- | --- |
| 9 direct-wired keys → **3×3 matrix** (3 rows + 3 cols on GP8–13, one 1N4148 per switch) | GP14–16 (3) |
| Display MISO unused (driver is write-only) → leave unrouted | GP28 (1) |

**Spent on new signals (4):**

| New signal | Pin |
| --- | --- |
| I2S data-in (BCLK/LRCLK shared with the DAC clocks) | GP16 |
| I2C SDA/SCL (codec ctrl + MAX17048) | GP14/15 |
| Codec MCLK (PIO-generated 256·fs) | GP28 |

**MCLK instead of jack detect** (decided during schematic capture): nearly
every affordable codec module (ES8388 etc.) requires an MCLK the PIO I2S
engine doesn't currently produce, so GP28 goes to MCLK — a codec that PLLs
from BCLK (TLV320AIC3204) can simply leave the pin NC. Headphone jack
detect and speaker auto-mute were dropped with it: the PJ311 jack footprint
this board standardizes on has no switch contact anyway, and the existing
manual kill switch (J9) already covers speaker muting. Result: **26/26
used, zero spare** — any future feature reopens this fight (the accepted
price of the non-XL module).

Matrix notes: diodes are mandatory — tracker key combos mean 3+ simultaneous
presses, and a diode per switch (9× 1N4148, same part as D1) makes the
matrix ghost-free. GP assignments above are indicative; final choice should
respect RP2350 I2C/PIO pin-function tables during schematic capture.

## 5. Workstreams

### A. PCB rev (KiCad) — the part that lives in this repo
1. New branch off `revised-pcb`. U1 is a **drop-in swap** (Pico LiPo 2 uses
   the standard Pico footprint) — verify USB-C connector overhang and the
   JST battery connector position against the sandwich clearances.
2. **Rewire the key block as a 3×3 matrix**: rows/cols on GP8–13, one
   1N4148 per switch (9×, same part as D1 — cheap THT). Key positions and
   caps unchanged; only the copper changes.
3. Display J7, board outline, display window, key positions, and MIDI jacks
   stay unchanged — the existing back panel STL survives with only mic-hole
   + HP-jack edits.
4. Replace U3 (PCM5102 6-pin header) with codec module/IC footprint; route
   I2S in/out + I2C; add mic bias/input and HP jack with detect switch.
5. Add 3.5 mm headphone jack (PJ-320 style to match existing MIDI jacks),
   MEMS mic footprint + acoustic port, MAX17048 + sense wiring to battery.
6. Update `pico_tracker.csv` BOM + README table; re-run ERC/DRC; regenerate
   gerbers.

### B. Firmware — the long pole (fork of xiphonics/picoTracker)
1. **RP2350 build target**: add `PICO_PLATFORM=rp2350` preset with the
   Pico LiPo 2 board file (`PICO_BOARD`); FreeRTOS RP2350 port already
   vendored; fix whatever `-Werror` fallout appears. (Small–medium.)
1b. **Key-matrix scan driver**: replace the 9-GPIO direct reads in
   `picoTrackerEventManager` with a 3×3 scan (drive one column low at a
   time, read 3 rows; full sweep every tick). With per-switch diodes, all
   9 keys are independently readable in any combination — ALT-holds and
   multi-key tracker combos behave identically to direct wiring; scanning
   at ≥1 kHz keeps added latency under 1 ms. Keep debounce and event
   semantics unchanged (upstream requires identical key behavior).
   (Small–medium.)
2. **PSRAM sample memory**: init RP2350 QMI PSRAM (8 MB), point the sample
   pool at it, replace/augment `LOAD_IN_FLASH`. This is the change that makes
   "more than a toy amount of sample memory" real. (Medium.)
3. **Codec driver**: I2C register setup + PIO I2S full-duplex (new PIO
   program for RX; TX program exists in `audio_i2s.pio`), plus a PIO/clock
   divider generating 256·fs MCLK on GP28 for codecs that need it (pick a
   sys-clock that divides to exact audio rates). Implement the real
   `record.h` API (`SetInputSource`, mic gain, levels) — the app-side
   `RecordStreamer` UI already exists upstream. (Medium–hard; the core new
   engineering.)
4. **Fuel gauge driver** feeding the existing battery-gauge view; upstream
   already has a TODO noting the pico's approximate metering
   (`View.cpp:415`). (Small.)
5. **8-channel perf validation** on RP2350 (M33 + FPU @150 MHz+, dual core).
   Upstream README says the RP2040 struggles with 8 channels; RP2350 should
   close most of that, but measure. (Test/tune.)
6. **Engage upstream early**: DEV.md invites per-case GitHub issues; keys,
   screenmap, and file formats must stay Advance-compatible. An RP2350 +
   codec "community hardware v3" may be upstreamable, or can live in
   `picoTracker-playground`. License is BSD-3 — no obstacles.

### C. Enclosure
- Back panel STL: mic port + HP jack cutout only; verify the ~11 mm internal
  depth still clears the codec module (prefer low-profile/SMD parts if not).

### D. Validation
- Bring-up checklist: power/charge → display → SD → keys → MIDI loopback →
  DAC playback → HP out → recording from line-in → mic → battery gauge.
- Burn-in: 8-channel project playback for CPU headroom; 6 h battery test.

## 5½. Implementation status: `kicad_advance/`

`kicad_advance/` is a copy of `kicad/` carrying the schematic + PCB updates:

- **Schematic — done** (ERC: 0 errors): key matrix (COL0-2 on GP11-13, ROW0-2
  on GP8-10, D2–D10), codec headers J11 `CODEC_DIG` (3V3/GND/MCLK/BCK/LRCK/
  DIN/DOUT/SDA/SCL) + J12 `CODEC_ANA` (LINE_L/R, MIC_IN, L_AMP/R_AMP, AGND),
  J13 line-in + J14 phones jacks, J15 mic module, J16 fuel-gauge header on
  I2C, MCLK on GP28, PCM5102 (old U3) removed. Amp input now taps the codec
  HP outs (nets `L_AMP`/`R_AMP` unchanged at J10).
- **PCB — placed and routed**: all 9 diodes sit on **B.Cu (back)** among the
  switch rows (cathode toward its ROW), codec headers J11/J12, jacks J13/J14,
  mic J15 and fuel-gauge J16 are placed on-board along the right, and all 47
  new-net connections (COL/ROW matrix, I2S BCK/LRCK/DIN/data-in, MCLK, I2C
  SDA/SCL, MIC_IN, L_AMP/R_AMP/AGND, and header VCC/GND) are auto-routed on
  a 0.2 mm grid (0.25 mm tracks, 0.25 mm clearance), with GND-island stitch
  vias and zones refilled. Total DRC count *dropped* 528 → 343 (the refill
  cleaned up stale-fill shorts in the original pour).
- **Verified clean on the new nets**: the only DRC item touching a new net is
  an I2S_SDIN↔VCC pad-proximity flag at U1.21/R4.2 that is **already present
  in the pre-routing board** (stranded R4.2 VCC pad in the dense MIDI/power
  cluster) — not introduced by this routing. Everything else new is
  cosmetic (silkscreen ref-designator clipping on the back-side parts).
- **Known pre-existing (not from this work)**: the original hand-routed
  MIDI/opto/power cluster around U1's south edge carries ~70 strict-DRC
  "shorts" (sub-0.2 mm hand spacing) and a stranded R4.2 VCC pad; these
  predate the conversion and were left untouched.
- **Conventions kept**: J13/J14 PJ311 pads are left unnetted like the
  existing MIDI jacks (symbol S/R/T vs pads 1–6) — hand-wire as before.

Remaining PCB polish (optional, human pass in KiCad): tidy the auto-router's
B.Cu detours, add GND thermal spokes on a few U1 pads flagged as 1-spoke,
and resolve the pre-existing MIDI-cluster strict-DRC shorts if a clean DRC
is wanted before fab.

## 6. Explicitly out of scope (and why)

- **720×720 HiDPI display**: no RP-family chip can drive it (no LTDC/MIPI-DSI,
  and the open firmware's renderer is a character-cell engine). Doing this
  means a new MCU class (STM32H7/i.MX RT + SDRAM + RGB/DSI panel) and a
  ground-up firmware port with no public reference — that's designing a
  second Advance, not converting this board.
- **Display upgrades generally (parked by decision)**: the best candidate if
  this is ever revisited is the **ST7796S 4.0" 480×320 SPI** — RGB565 over
  SPI like the ILI9341, 1:1 rewire of J7, needing a new init sequence, 15×13
  px `chargfx` cells, a redrawn ~12×12 font, and a larger board
  outline/back panel. Deferred to keep this rev's scope on audio + memory.
- **480×480 square RGB panel (Presto-style)**: considered and parked. Proven
  on RP2350B (Pimoroni Presto drives an ST7701 over parallel RGB via
  PIO+DMA+PSRAM framebuffer on core1), but costs ~20 GPIO, a core, and a
  much bigger driver effort than the chosen SPI panel. Revisit only after
  the codec/PSRAM work lands, if ever.
- **48 MB**: 8 MB PSRAM is the practical RP2350 ceiling per QMI CS; it's a
  30× upgrade over today and covers most real use.
- Matching the Advance's industrial design/size.

## 7. Effort & cost (rough)

| Item | Estimate |
| --- | --- |
| PCB rev + layout + BOM | ~2–4 evenings (U1 drop-in; key-matrix rewiring + audio section are the changes) |
| Fab + parts per unit | board ~$5–15; Pico LiPo 2 ~$14; codec $2–10; gauge ~$2; jack/mic ~$3; 9× 1N4148 pennies — **≈ $55–75/unit** with existing display/keys/amp |
| Firmware: RP2350 target | days |
| Firmware: key-matrix scan driver | days |
| Firmware: PSRAM samples | ~1 week part-time |
| Firmware: codec + recording | 1–3 weeks part-time (long pole) |
| Firmware: fuel gauge, polish | days |

## 8. Open questions / risks

1. **Pico LiPo 2 (PIM775) details** — from the schematic, confirm: PSRAM CS
   is fully internal; GP43 battery-sense divider ratio (for a firmware
   fallback read); which GPIO the Qw/ST connector shares (if it lands on
   pins this design uses, the connector is simply unusable — fine, the
   I2C bus is on GP14/15 anyway); USB-C/JST mechanical overhang vs the
   sandwich enclosure.
2. **Codec choice** — module (ES8388, easy, matches THT philosophy) vs chip
   (AIC3204, cleaner, SMD reflow). Decide based on how SMD-averse this board
   wants to stay (it just moved *to* THT).
3. **PIO budget** — SDIO uses pio1, audio TX uses pio0; RX I2S needs a free
   state machine (pio0 has 3 spare; RP2350 adds pio2 — fine, but verify DMA
   channel allocation).
4. **Upstream appetite** — whether xiphonics accepts an RP2350/codec target
   or it lives as a community fork determines long-term maintenance cost.
5. **Advance feature drift** — Advance firmware is versioned separately;
   "equivalence" is a moving target. Anchor on: sampling, 8 channels, HP out,
   USB+TRS MIDI, real battery meter.

## Sources

- [xiphonics picoTracker product page](https://xiphonics.com/products/picotracker)
- [xiphonics/picoTracker firmware](https://github.com/xiphonics/picoTracker) (local clone examined: `sources/Adapters/picoTracker/platform/gpio.h`, `audio/record.h`, `docs/DEV.md`, `CMakePresets.json`)
- [picoTracker Advance manual](https://manual.xiphonics.com/advance/introduction.html)
- [Elektronauts Advance thread](https://www.elektronauts.com/t/xiphonics-picotracker-advance/243649)
- [Pimoroni Pico LiPo 2](https://shop.pimoroni.com/en-us/products/pimoroni-pico-lipo-2) · [Pico LiPo 2 XL W](https://shop.pimoroni.com/en-us/products/pimoroni-pico-lipo-2-xl-w)
- [Pimoroni Presto](https://shop.pimoroni.com/en-us/products/presto) + [ST7701 driver internals](https://deepwiki.com/pimoroni/presto/2.2-st7701-display-driver) — existence proof for the parked 480×480 RGB option
