# ES8388 Module — Hand-drawing the schematic in KiCad

A step-by-step, follow-along guide to draw the codec module schematic yourself in Eeschema
(KiCad 9/10). Values and connections come from `ES8388_MODULE_PLAN.md` (which was verified against
the mainboard netlist). Work top-to-bottom; there are **checkpoints** where you can stop and sanity-check.

## Eeschema hotkeys you'll use constantly
| Key | Action |
|---|---|
| `A` | Add symbol |
| `P` | Add **power** symbol (GND, +3V3, PWR_FLAG) |
| `W` | Draw wire |
| `L` | Place **net label** (names a wire) |
| `M` / `G` | Move / drag (G keeps wires attached) |
| `R` | Rotate |
| `C` | Duplicate (copy) the hovered item |
| `E` | Edit properties/value |
| `Q` | Place **no-connect** (×) flag on unused pins |
| `Del` | Delete |
| `Ctrl+S` | Save |

Tip: you almost never need long wires. Put a short stub on a pin and give it a **label** (`L`);
two stubs with the same label are the same net. We'll use that heavily.

---

## Phase 0 — Get the ES8388 symbol + footprint

KiCad has no built-in ES8388. Easiest path that also guarantees the footprint matches the JLC part
(LCSC **C365736**) you'll assemble:

```bash
pip install easyeda2kicad
easyeda2kicad --full --lcsc_id=C365736 --output ~/kicad-libs/es8388
```

This produces `es8388.kicad_sym`, an `es8388.pretty/` footprint folder, and a 3D model.

In KiCad: **Preferences → Manage Symbol Libraries → Project Specific Libraries → +**, add
`~/kicad-libs/es8388.kicad_sym`. Do the same under **Manage Footprint Libraries** for the `.pretty`.

The two pin headers need no import — they use stock KiCad symbols/footprints (Phase 6).

> Alternative if you'd rather not use easyeda2kicad: make a symbol by hand in the Symbol Editor —
> a 28-pin rectangle + EP pin, names exactly as in the pin table below. Slower; the importer is worth it.

**Checkpoint 0:** you can type `A`, search "ES8388", and place the symbol.

### Parts list — set each symbol's Value + LCSC as you place it
All JLCPCB **Basic/Preferred** (no assembly fee) except the ES8388. In KiCad you can stuff the LCSC
code into a field named `LCSC` (or `JLCPCB`) on each symbol so the assembly BOM is ready later.

| Value / part | Qty | Pkg | LCSC | Use |
|---|---|---|---|---|
| ES8388 | 1 | QFN‑28‑EP 4×4 | **C365736** | codec (U1) |
| 100 nF X7R 16V | ~8 | 0402 | **C1525** | every 0.1 µF decoupling |
| 1 µF X5R 25V | 5 | 0402 | **C52923** | input coupling (LINE_L/R, MIC) + aux line-out (LOUT2/ROUT2) |
| 4.7 µF X5R 16V | 4 | 0603 | **C19666** | DVDD + VREF/VMID/ADCVREF |
| 10 µF X5R 10V | 2 | 0603 | **C19702** | +3V3 / +3V3A bulk |
| 100 µF X5R 6.3V | 4 | 1206 | **C15008** | HPVDD **2× parallel** + 2× HP DC‑block |
| 30 pF C0G 50V | 1 | 0402 | **C1570** | optional MCLK edge cap |
| 2.2 nF X7R 50V | 1 | 0402 | **C1531** | optional I²C‑clock RC |
| 33 Ω | 5 | 0402 | **C25105** | digital series damping |
| 22 Ω | 2 | 0402 | **C25092** | LOUT1/ROUT1 series |
| 10 Ω | 1 | 0402 | **C25077** | AVDD↔HPVDD |
| 0 Ω | 1 | 0402 | **C17168** | CE→GND (addr 0x20) |
| 5.1 kΩ | 2 | 0402 | **C25905** | I²C pull‑ups (4.7 kΩ alt = C23162, 0603) |
| 330 Ω | 1 | 0402 | **C25104** | optional I²C‑clock series |
| Ferrite 600 Ω@100 MHz | 1 | 0603 | **C1002** | FB1 analog‑rail filter |
| 1×9 male header 2.54 mm | 1 | THT | — | J1 (mates J11) |
| 1×6 male header 2.54 mm | 1 | THT | — | J2 (mates J12) |
| 1×3 male header 2.54 mm | 1 | THT | — | J3 aux line-out (LOUT2/ROUT2/GND) |

---

## Phase 1 — New project

1. **File → New Project** → name it `kicad_es8388_module`, save it in the repo root next to
   `kicad_advance/`.
2. Open the `.kicad_sch`. You'll be on one sheet — the whole module fits on one page.
3. Suggested canvas zones (just so it stays readable):
   - **Left:** power in + supply/decoupling network
   - **Center:** the ES8388
   - **Upper-right:** digital header `J1` + series resistors + I²C
   - **Lower-right:** analog header `J2` + coupling caps

---

## Phase 2 — Place the ES8388 and the ground/power symbols

1. `A` → search **ES8388** → place it in the center.
2. `E` on it → set **Reference = U1**.
3. `P` → place a **GND** symbol somewhere central (you'll copy it around with `C`).
4. `P` → place a **+3V3** symbol (this is our digital rail = header VCC).
5. `P` → place a second power symbol, `E` its value to **+3V3A** (analog rail; we'll make it from
   +3V3 through a ferrite bead). If +3V3A isn't offered, use a generic power symbol and rename it.

Here is the ES8388 pin reference you'll wire against (QFN-28 + EP):

| Pin | Name | Pin | Name |
|----:|------|----:|------|
| 1 | MCLK | 15 | LOUT2 |
| 2 | DVDD | 16 | HPVDD |
| 3 | PVDD | 17 | AVDD |
| 4 | DGND | 18 | AGND |
| 5 | SCLK | 19 | ADCVREF |
| 6 | DSDIN | 20 | VMID |
| 7 | LRCK | 21 | RIN2 |
| 8 | ASDOUT | 22 | LIN2 |
| 9 | NC | 23 | RIN1 |
| 10 | VREF | 24 | LIN1 |
| 11 | ROUT1 | 25 | NC |
| 12 | LOUT1 | 26 | CE |
| 13 | HPGND | 27 | CDATA |
| 14 | ROUT2 | 28 | CCLK |
|    |      | 29 | EP (GND) |

**Checkpoint 2:** U1 placed, one each of GND / +3V3 / +3V3A power symbols on the sheet.

---

## Phase 3 — Power & decoupling network (left zone)

We split the 3.3 V into a clean analog rail. All caps go from the pin to **GND**.

**Rail generation:**
1. Place an **inductor/ferrite** symbol (`A` → `FerriteBead` or `Device:L`), Ref **FB1**, value e.g.
   `600R@100MHz`. Wire **+3V3 → FB1 → +3V3A**.
2. Place a **10 Ω** resistor `R1`. Wire **+3V3A → R1 → HPVDD (pin 16)**. (App-note isolation between
   AVDD and HPVDD.)

**Decoupling (place cap, `E` to set value, wire pin→cap→GND):**

**Supply pins** — wire to the rail **and** add the caps to GND:

| Codec pin | Wire to rail | Caps to GND |
|---|---|---|
| DVDD (2) | +3V3 | 4.7 µF ∥ 0.1 µF |
| PVDD (3) | +3V3 | 0.1 µF |
| AVDD (17) | +3V3A | 0.1 µF |
| HPVDD (16) | +3V3A via 10 Ω | **2× 100 µF** ∥ 0.1 µF (parallel — offsets 6.3 V MLCC DC-bias derating) |

**Reference pins** — these are voltages the chip generates *internally*. **Do NOT wire them to any
rail.** The only connection is a decoupling cap to GND (a smoothing tank on an internal bias voltage):

| Codec pin | Wire to rail | Caps to GND |
|---|---|---|
| VREF (10) | none — internal output | 4.7 µF ∥ 0.1 µF |
| VMID (20) | none — internal output | 4.7 µF ∥ 0.1 µF |
| ADCVREF (19) | none — internal output | 4.7 µF ∥ 0.1 µF |

**Grounds:** wire **DGND(4), AGND(18), HPGND(13), and EP(29)** all to **GND** symbols. (On the module,
one GND net is correct — the module is the single AGND↔GND star point; the analog/digital split is a
layout concern later, not a schematic-net concern.)

Reference designator tip: don't hand-number caps — you'll run **Tools → Annotate** at the end and KiCad
assigns C1, C2, … automatically. Just place them with the right values.

**Checkpoint 3:** every power pin (2,3,16,17) has decoupling; the three bias pins (10,19,20) have their
4.7 µF∥0.1 µF; all grounds tied. This is the fiddliest phase — take your time.

---

## Phase 4 — Digital header J1 + series resistors + I²C (upper-right)

1. Place the digital header last (Phase 6) or now — either works. For now, wire the codec side using
   **labels** so you don't need the header physically yet.

**Series-damping resistors (33 Ω each).** For each line: put a resistor, wire the codec pin to one
end, and put a **label** (`L`) on the *other* end with the header-facing net name:

| Codec pin | Series R | Label on far end |
|---|---|---|
| MCLK (1) | 33 Ω | `MCLK` |
| SCLK (5) | 33 Ω | `BCK` |
| LRCK (7) | 33 Ω | `LRCK` |
| DSDIN (6) | 33 Ω | `DIN` |
| ASDOUT (8) | 33 Ω | `I2S_SDIN` |

(Optional: a 30 pF cap from the **codec side** of the MCLK resistor to GND.)

**I²C:**
1. CDATA (27): wire a stub, label it `SDA`.
2. CCLK (28): optional 330 Ω series toward the header; label the header side `SCL`. (Optional 2.2 nF
   from the CCLK pin to GND for the app-note RC filter.)
3. **Pull-ups (populate these — the mainboard has none):** two **5.1 kΩ** resistors (C25905 — the
   Basic 0402 value; 4.7 kΩ isn't Basic in 0402), each from `SDA` and `SCL` up to **+3V3**. Put them
   on the header side of any series R.

**CE / I²C address (pin 26):**
- Place a **0 Ω** resistor `R?` from **CE (26) → GND** → address **0x20** (default).
- Optionally add a second pad/resistor from CE → +3V3 (leave unpopulated) for **0x22**. Never wire CE
  to a GPIO.

**NC pins:** press `Q` on pins **9** and **25** to drop no-connect flags.

**Checkpoint 4:** the 5 digital lines each go codec → 33 Ω → labeled net; SDA/SCL labeled with 5.1 kΩ
pull-ups to +3V3; CE has its 0 Ω to GND; NC flags on 9 & 25.

---

## Phase 5 — Analog inputs & outputs + coupling (lower-right)

All analog signals are AC-coupled. Use **labels** for the header-facing nets again.

**Inputs (label → coupling cap → codec pin):**

| Header net (label) | Coupling | Codec pin |
|---|---|---|
| `LINE_L` | 1 µF | LIN2 (22) |
| `LINE_R` | 1 µF | RIN2 (21) |
| `MIC_IN` | 1 µF | LIN1 (24) |

- The mic is mono → tie the unused **RIN1 (23)** to **GND** (or via 0.1 µF); it's disabled in registers.

**Outputs (codec pin → 22 Ω → DC-block 100 µF → label). Electrolytic polarity: `+` toward the codec
(VMID-biased) side:**

| Codec pin | Series | DC-block (+ to codec) | Header net (label) |
|---|---|---|---|
| LOUT1 (12) | 22 Ω | 100 µF | `L_AMP` |
| ROUT1 (11) | 22 Ω | 100 µF | `R_AMP` |

**Aux line-out (LOUT2/ROUT2)** — broken out on a separate header so the module is reusable outside
the picoTracker. Codec pin → 1 µF (line-level coupling) → aux label:

| Codec pin | Coupling | Aux label |
|---|---|---|
| LOUT2 (15) | 1 µF | `LINEOUT_L` |
| ROUT2 (14) | 1 µF | `LINEOUT_R` |

- Place **J3** = `Conn_01x03`: pin 1 `LINEOUT_L`, pin 2 `LINEOUT_R`, pin 3 `GND`.
  Footprint later = `PinHeader_1x03_P2.54mm_Vertical`. J3 is unused when mated to the picoTracker.
- (Optional: add a 22 Ω series on each like the main outputs — app-note style. Not required for a
  high-impedance line load.)

**Now only pins 9 & 25 are no-connects** (LOUT2/ROUT2 are used) — leave the `Q` flags on 9 & 25 only.

**Checkpoint 5:** three inputs coupled in (RIN1 grounded); two main outputs (LOUT1/ROUT1) via 22 Ω +
100 µF to L_AMP/R_AMP; two aux line-outs (LOUT2/ROUT2) via 1 µF to J3; only pins 9 & 25 no-connected.

---

## Phase 6 — The headers (the mating interface + aux)

1. `A` → **Conn_01x09** → place as **J1** (digital). `A` → **Conn_01x06** → place as **J2** (analog).
   `A` → **Conn_01x03** → place as **J3** (aux line-out).
2. Wire each header pin to a short stub and `L` a label matching the tables below. Same label name =
   connected to the matching net you already drew — that's how the header ties into the codec side.

**J1 (digital, 1×9):**
```
1 +3V3   2 GND   3 MCLK   4 BCK   5 LRCK   6 DIN   7 I2S_SDIN   8 SDA   9 SCL
```
**J2 (analog, 1×6):**
```
1 LINE_L   2 LINE_R   3 MIC_IN   4 L_AMP   5 R_AMP   6 AGND
```
**J3 (aux line-out, 1×3 — not part of the picoTracker mate):**
```
1 LINEOUT_L   2 LINEOUT_R   3 GND
```

- For J2 pin 6, label it `AGND` **but** also connect module GND here — simplest is to just label it
  `GND` so it merges into the module ground (the module is the star point). If you prefer to keep the
  name, label it `AGND` and drop a short wire joining `AGND` to a `GND` symbol once, near U1's EP.

> ⚠️ The **pin order above is fixed** — it's what the mainboard's J11/J12 expect. Physical 5.70 mm
> spacing between the two headers is set in the PCB layout step, not here.

**Checkpoint 6:** J1 has 9 labeled pins, J2 has 6, J3 has 3 (LINEOUT_L/R + GND), every label name
matches one already used on the codec side.

---

## Phase 7 — Footprints, ERC, and finish

1. **PWR_FLAG:** `P` → place **PWR_FLAG** and attach one to **+3V3** and one to **GND** (power enters
   from a connector, so ERC needs these or it complains about "no driving source"). Optionally one on
   +3V3A too.
2. **Tools → Annotate Schematic** → fills in R/C/etc. reference numbers.
3. **Assign footprints** (Tools → Assign Footprints, or the property on each symbol):
   - U1 → the imported `es8388` footprint (QFN-28-EP 4×4).
   - J1 → `Connector_PinHeader_2.54mm:PinHeader_1x09_P2.54mm_Vertical`
   - J2 → `Connector_PinHeader_2.54mm:PinHeader_1x06_P2.54mm_Vertical`
   - J3 → `Connector_PinHeader_2.54mm:PinHeader_1x03_P2.54mm_Vertical`
     (these are the exact footprints the mainboard uses — guarantees the mate.)
   - Passives → the packages in the parts list (0402 small, 0603 for 4.7/10 µF, **1206** for the
     100 µF C15008 MLCCs).
4. **Inspect → Electrical Rules Checker → Run.** Resolve:
   - Unconnected pins → add `Q` no-connects (NC pins 9, 25 only — LOUT2/ROUT2 now go to J3).
   - "No driving source" on power → add the PWR_FLAGs from step 1.
5. `Ctrl+S`.

**Done.** Next you'd do **Tools → Update PCB from Schematic** and start the layout (that's where the
5.70 mm header spacing, ground pours, and the AGND↔GND EP star point get placed).

---

## Quick full connection recap (for wiring without scrolling)

```
POWER
 +3V3 (=VCC, J1.1) ── DVDD(2)[4.7u+0.1u]  ── PVDD(3)[0.1u]  ── 5.1k pull-ups(SDA,SCL)
 +3V3 ──FB1── +3V3A ── AVDD(17)[0.1u]
 +3V3A ──10R── HPVDD(16)[2×100u+0.1u]
 GND ── DGND(4), AGND(18), HPGND(13), EP(29), J1.2, J2.6
 VREF(10)[4.7u+0.1u]  VMID(20)[4.7u+0.1u]  ADCVREF(19)[4.7u+0.1u]

DIGITAL (each via 33R)         I2C
 MCLK  ─33R─ MCLK(1)            SDA ── CDATA(27), 5.1k→+3V3
 BCK   ─33R─ SCLK(5)            SCL ─(330R opt)─ CCLK(28), 5.1k→+3V3, 2.2n→GND opt
 LRCK  ─33R─ LRCK(7)            CE(26) ─0R─ GND   (addr 0x20)
 DIN   ─33R─ DSDIN(6)           NC: pins 9, 25 → no-connect
 I2S_SDIN ─33R─ ASDOUT(8)

ANALOG
 LINE_L ─1u─ LIN2(22)           LOUT1(12) ─22R─100u(+ to codec)─ L_AMP
 LINE_R ─1u─ RIN2(21)           ROUT1(11) ─22R─100u(+ to codec)─ R_AMP
 MIC_IN ─1u─ LIN1(24)           LOUT2(15) ─1u─ LINEOUT_L (J3.1)
 RIN1(23) → GND (mic mono)      ROUT2(14) ─1u─ LINEOUT_R (J3.2)
                                NC: pins 9, 25 only
```
