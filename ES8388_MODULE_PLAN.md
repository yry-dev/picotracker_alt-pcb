# ES8388 Codec Module — Design Plan

A small daughterboard that carries the **ES8388** SMD audio codec (QFN‑28) plus all its
support passives, and plugs directly into the two codec headers already on the
picoTracker Advance mainboard (`J11 CODEC_DIG`, `J12 CODEC_ANA`). This keeps the fine‑pitch
QFN and sensitive analog network off the hand‑assembled mainboard and turns the codec into a
solder‑and‑go module, exactly like the reference board in `~/Desktop/example.png`.

> **Sources for this plan**
> - ES8388 User Guide / App Note (pcbartists.com) — §3 Typical Application Circuit, §2 operating
>   conditions, §4 I²C addressing, §5 digital audio interface, layout notes.
> - `~/Desktop/example.png` — a community ES8388 breakout (2×10 headers) — used for the
>   physical concept only; **our pinout is different** (it must match our mainboard).
> - The Advance mainboard itself: header pinouts and analog net destinations were read
>   directly from `kicad_advance/pico_tracker.kicad_sch` / `.kicad_pcb`.

---

## 1. Goal & constraints

| Requirement | Decision |
|---|---|
| Must plug into existing `J11`/`J12` with **no mainboard changes** | Module carries two **male** pin headers matching J11 (9‑pin) and J12 (6‑pin) exactly |
| Match the mainboard's net names/order | Pinouts frozen below, read from the schematic |
| Codec supply | Header `VCC` = Pico **3.3 V** rail (confirmed: Pico 2 `3V3` net). Digital runs off it; analog is **ferrite‑bead‑isolated 3.3 V** on‑module (no LDO — can't regulate 3.3→3.3) |
| Assembly | ES8388 + 0402/0603 passives are JLCPCB‑assembled on the module; mainboard stays simple |
| Cost/availability | ES8388 = LCSC **C365736**, ~$1.10, 14k–24k stock (chosen over TLV320AIC3204: cheaper + far more common) |

---

## 2. Mating interface (frozen — this is the whole point of the module)

Read from `kicad_advance/pico_tracker.kicad_pcb`. Both headers are single‑row **2.54 mm** pitch,
pins running in +Y, sitting side‑by‑side:

- `J11 CODEC_DIG` — 9 pins, column at **X = 104.80**, pin 1 at **Y = 91.84**
- `J12 CODEC_ANA` — 6 pins, column at **X = 110.50**, pin 1 at **Y = 91.65**
- **Inter‑column spacing ΔX = 5.70 mm**, pin‑1 rows aligned within 0.19 mm.

> ⚠️ 5.70 mm is *not* an integer 2.54 multiple, so **do not eyeball it**. The safe path:
> import the exact `J11`/`J12` footprints/positions from the mainboard PCB into the module so
> the geometry is guaranteed identical, then populate male headers.

### J11 `CODEC_DIG` — digital (9‑pin)
Pico GPIO confirmed from the exported netlist (`U1` = Pico 2):

| Pin | Mainboard net | Pico GPIO | ES8388 pin | Notes |
|----:|---------------|-----------|-----------|-------|
| 1 | **VCC** (3.3 V) | Pico `3V3` | DVDD/PVDD feed | digital supply in |
| 2 | **GND** | — | DGND | |
| 3 | **MCLK** | **GP28** | MCLK (1) | PIO 256·fs |
| 4 | **BCK** | **GP18** | SCLK (5) | I²S bit clock |
| 5 | **LRCK** | **GP19** | LRCK (7) | I²S word clock |
| 6 | **DIN** | **GP17** | DSDIN (6) | DAC data, Pico → codec |
| 7 | **I2S_SDIN** | **GP16** | ASDOUT (8) | ADC data, codec → Pico |
| 8 | **SDA** | **GP14** | CDATA (27) | I²C data |
| 9 | **SCL** | **GP15** | CCLK (28) | I²C clock |

### J12 `CODEC_ANA` — analog (6‑pin)
Destinations verified from the mainboard (which jack/amp each net actually reaches):

| Pin | Mainboard net | Goes to on mainboard | ES8388 side |
|----:|---------------|----------------------|-------------|
| 1 | **LINE_L** | `J13 LINE_IN` jack (input) | → **LIN2 (22)** line input |
| 2 | **LINE_R** | `J13 LINE_IN` jack (input) | → **RIN2 (21)** line input |
| 3 | **MIC_IN** | `J15 MIC_MODULE` (electret) | → **LIN1/RIN1 (24/23)** mic input |
| 4 | **L_AMP** | `J14 PHONES` jack **and** `J10 DclassAMP` | ← **LOUT1 (12)** output |
| 5 | **R_AMP** | `J14 PHONES` jack **and** `J10 DclassAMP` | ← **ROUT1 (11)** output |
| 6 | **AGND** | analog ground | AGND/HPGND |

Key takeaways that drive the circuit:
- **LINE_L/LINE_R are codec *inputs*** (from the line‑in jack), not outputs.
- **L_AMP/R_AMP are codec *outputs*** feeding both the headphone jack and the class‑D
  speaker amp — so they need real drive capability → use the **HP driver LOUT1/ROUT1**.
- **MIC_IN is a single‑ended** mic signal from the existing mic module.

---

## 3. Power architecture

Header gives us one 3.3 V rail + ground. Split on‑module into clean digital/analog:

```
VCC(3.3V) ─┬─ DVDD (pin2)  ── 4.7µF ∥ 0.1µF ──GND
           ├─ PVDD (pin3)  ── 0.1µF ──GND        (I²C buffer supply)
           └─[FB ferrite]─┬─ AVDD  (pin17) ── 0.1µF ──GND
              +10µF bulk   ├─[10Ω]─ HPVDD (pin16) ── 2×100µF ∥ 0.1µF ──GND
                           └─ analog 3.3V_A reference island
```

- App note recommends a dedicated LDO for the analog side; since our header only exposes
  3.3 V we instead use a **ferrite bead + bulk cap** LC filter (standard for these modules).
- **10 Ω between AVDD and HPVDD** (app‑note recommended when they share a rail).
- One solid **AGND pour**; single‑point tie to DGND under the QFN EP.

*(If a future mainboard rev exposes 5 V on the header, an LM1117‑3.3 per the app note's typical
circuit would give marginally lower noise — noted as optional, not required.)*

---

## 4. Codec support circuit (values from the app‑note Typical Application Circuit)

**Reference / bias caps (all to GND, close to the pin):**
| Pin | Net | Cap |
|---|---|---|
| 10 | VREF | 4.7µF ∥ 0.1µF |
| 20 | VMID | 4.7µF ∥ 0.1µF |
| 19 | ADCVREF | 4.7µF ∥ 0.1µF |
| 16 | HPVDD | **2× 100µF** ∥ 0.1µF |
| 17 | AVDD | 0.1µF |
| 2 | DVDD | 4.7µF ∥ 0.1µF (behind ferrite from VCC) |
| 3 | PVDD | 0.1µF |

**Digital lines — series damping resistors on every clock/data line (app note uses 33 Ω;
22 Ω also fine):**
- MCLK(1), SCLK(5), DSDIN(6), LRCK(7), ASDOUT(8): **33 Ω series** each.
- MCLK optionally **30 pF to GND** for edge rate.

**I²C (CCLK/CDATA, pins 28/27):**
- **Pull‑ups to 3.3 V on both — POPULATE them on the module.** Netlist trace shows the
  **mainboard has *no* I²C pull‑ups** (SCL/SDA only reach Pico GP15/GP14, codec J11, and the
  fuel‑gauge J16 — no resistors). The bus needs pull‑ups and the codec module is the right owner.
- Use **4.7 kΩ** (not the app note's 1 kΩ) so that *if* the MAX17048 fuel‑gauge module also carries
  pull‑ups, the parallel value stays sane. Put them on a solder‑jumper so they can be lifted.
- App‑note **R‑C low‑pass on the clock**: 330 Ω series + ~2.2 nF to GND (optional; improves noise).

**CE / I²C address (pin 26):**
- **0 Ω to DGND → address `0x20`** (default). Pull‑up option pad to PVDD → `0x22`.
- Expose as a solder‑jumper so firmware address is selectable. **Never** wire CE to a GPIO.

**Analog I/O coupling:**
| Path | Pins | Coupling |
|---|---|---|
| Line‑out (to L_AMP/R_AMP → phones + amp) | LOUT1(12)/ROUT1(11) | 22 Ω series + **100 µF** DC‑block (elec.), + ~330 Ω bleed |
| Line‑in (from LINE_L/LINE_R jack) | LIN2(22)/RIN2(21) | 1 µF series AC‑couple + 0.1µF/47µF filter per app note |
| Mic‑in (from MIC_IN) | LIN1(24)/RIN1(23) | 1 µF AC‑couple; single‑ended (tie unused input to VMID via 0.1µF) |

> **DC‑block placement — RESOLVED (was Open Q1):** netlist confirms `L_AMP` = {J12.4, J14.T
> (phones), J10.4 (amp)} and `R_AMP` = {J12.5, J14.R, J10.7} are **bare 3‑node nets with zero
> series components** on the mainboard. The ES8388 HP outputs are VMID‑biased, so the **module
> must provide the series DC‑block caps** (100 µF). Confirmed as plan of record.

---

## 5. Bill of materials (module)

| Value / part | Qty | Pkg | LCSC | Lib | Use |
|---|---|---|---|---|---|
| ES8388 | 1 | QFN‑28‑EP 4×4 | **C365736** | Extended | codec |
| 100 nF X7R 16V | ~8 | 0402 | **C1525** | Basic | all 0.1µF decoupling |
| 1 µF X5R 25V | 5 | 0402 | **C52923** | Basic | input coupling (LINE_L/R, MIC) + aux line-out (LOUT2/ROUT2) |
| 4.7 µF X5R 16V | 4 | 0603 | **C19666** | Basic | DVDD + VREF/VMID/ADCVREF |
| 10 µF X5R 10V | 2 | 0603 | **C19702** | Basic | +3V3 / +3V3A bulk |
| 100 µF X5R 6.3V | 4 | 1206 | **C15008** | Basic | **HPVDD 2× parallel** + 2× HP DC‑block |
| 30 pF C0G 50V | 1 | 0402 | **C1570** | Preferred | optional MCLK edge cap |
| 2.2 nF X7R 50V | 1 | 0402 | **C1531** | Preferred | optional I²C‑clock RC |
| 33 Ω 1% | 5 | 0402 | **C25105** | Basic | digital series damping |
| 22 Ω 1% | 2 | 0402 | **C25092** | Basic | LOUT1/ROUT1 series |
| 10 Ω 1% | 1 | 0402 | **C25077** | Basic | AVDD↔HPVDD isolation |
| 0 Ω jumper | 1 | 0402 | **C17168** | Basic | CE→GND (I²C addr 0x20) |
| 5.1 kΩ 1% | 2 | 0402 | **C25905** | Basic | I²C pull‑ups (4.7 kΩ alt = C23162, 0603) |
| 330 Ω 1% | 1 | 0402 | **C25104** | Basic | optional I²C‑clock series |
| Ferrite 600 Ω@100 MHz, 200 mA | 1 | 0603 | **C1002** | Basic | FB1 analog‑rail filter |
| 1×9 male header 2.54 mm | 1 | THT | — | — | J1, mates mainboard J11 |
| 1×6 male header 2.54 mm | 1 | THT | — | — | J2, mates mainboard J12 |
| 1×3 male header 2.54 mm | 1 | THT | — | — | J3, aux stereo line-out (LOUT2/ROUT2/GND); unused on picoTracker |

All passives are JLCPCB **Basic/Preferred** (no assembly fee); only the ES8388 is Extended (one +$3
fee). Headers are through‑hole — usually hand‑soldered (THT assembly adds cost). The 0.1 µF count is
approximate — one per decoupled pin; annotate in KiCad to get the exact number.

> **Why 2× 100 µF on HPVDD:** C15008 is a 6.3 V X5R MLCC and loses a large fraction of its rated
> capacitance under DC bias — at 3.3 V it behaves like roughly half its face value. Two in parallel
> restores an effective ~100 µF of bulk on the headphone supply. (The two HP‑output DC‑block 100 µF
> caps sit at low DC bias, so a single each is fine.)

---

## 6. PCB & mechanical

- **2‑layer**, ~**25 × 22 mm** (fits QFN + passives + two header columns; comparable to the
  example board). Top: QFN + digital passives; keep analog passives clustered by the analog pins.
- **Header placement:** J1/J2 at the imported mainboard coordinates (ΔX = 5.70 mm, pin‑1 aligned).
  Headers along one edge so the module stands over the mainboard like the example.
- **Orientation / pin‑1:** module is a plug‑in daughterboard — its pad (i) must sit directly over
  mainboard socket (i). Import footprints, then **verify pin‑1 dots line up** in a 3D/overlay check
  before ordering (classic mirror‑flip bug).
- **Grounding — the module is the star point (netlist‑confirmed):** on the mainboard `AGND` and
  `GND` are **separate nets**. `AGND` = {J12.6 codec, J13/J14 jack sleeves, J10.5/6 amp}; `GND` =
  {J11.2 codec, Pico, SD, mic, fuel gauge, …}. The codec module receives *both* (GND on J11.2,
  AGND on J12.6) and is the natural place to **tie AGND↔GND at the QFN EP thermal pad** (single
  star point). Continuous AGND pour under the analog section, DGND under the digital section,
  joined only at the EP; EP stitched down with a via array.
- ⚠️ **Ground‑loop watch:** the class‑D amp module (`J10`) is the *only other* connector touching
  both AGND (5,6) and GND (2), so it may also bridge them internally → a potential loop with the
  codec module's star tie. Prefer the codec EP as the single tie; if the amp module hard‑ties them
  too, that's an acceptable‑but‑noted second bridge (class‑D input is not sensitive).
- **Layout notes from app note:** decoupling caps hard against their pins; if a future differential
  mic is used, route MIC_INP/MIC_INN as a parallel pair. Keep MCLK away from analog traces.
- Silkscreen: label both headers with net names + pin 1, and mark the CE address jumper
  (`0x20` default) and the DNP I²C pull‑ups.

---

## 7. Open questions — RESOLVED by netlist trace

Traced against the KiCad‑exported netlist of `kicad_advance/pico_tracker.kicad_sch`. Results:

1. **HP output DC‑block → module provides it.** `L_AMP`/`R_AMP` are bare nets, codec → phones jack →
   amp, no series parts. Module carries the 100 µF DC‑block caps. ✅ resolved.
2. **I²C pull‑ups → module provides them.** Mainboard has **no** SCL/SDA pull‑ups anywhere. Populate
   4.7 kΩ pull‑ups on the module (jumpered). ✅ resolved. *(Only residual check: whether the plug‑in
   MAX17048 fuel‑gauge module carries its own pull‑ups — if so they parallel the module's; harmless
   at 4.7 kΩ.)*
3. **MCLK → confirmed GP28.** `MCLK` = {J11.3, U1.34 `GPIO28_ADC2`}. Firmware/PIO must generate
   256·fs there; ES8388 slave mode auto‑detects the ratio → **no crystal on module.** ✅ resolved.
4. **Mic → single‑ended, AC‑couple on module.** `MIC_IN` = {J12.3, J15.3}; `J15 MIC_MODULE` is a
   **powered 3‑pin module** (VCC/GND/signal), so it outputs a biased/buffered signal. Module
   AC‑couples it (1 µF) into LIN1/RIN1; unused input tied to VMID. ✅ resolved.

**New finding (grounding):** `AGND` and `GND` are **separate nets** on the mainboard, bridged only
through plug‑in modules. The codec module must be the AGND↔GND **star point** at the QFN EP — see §6
(and the amp‑module ground‑loop note).

Full I²S/I²C GPIO map (netlist‑verified): DIN=GP17, ASDOUT/I2S_SDIN=GP16, BCK=GP18, LRCK=GP19,
MCLK=GP28, SDA=GP14, SCL=GP15.

---

## 8. Suggested build sequence

1. ~~Confirm Open Questions~~ — done (§7). Only residual: does the MAX17048 fuel‑gauge module carry
   its own I²C pull‑ups (harmless either way at 4.7 kΩ).
2. New KiCad project `kicad_es8388_module/`. Import `J11`/`J12` footprints + relative placement from
   `kicad_advance`.
3. Draw schematic from §2/§4 tables; run ERC.
4. Place QFN central, passives by domain, headers on the mating edge; pour grounds; DRC.
5. 3D‑overlay the module against the mainboard to verify header alignment + no component collision
   with mainboard parts in the stack‑up gap.
6. Generate JLCPCB fab + assembly outputs (the repo already has a fabrication‑toolkit config to model on).
