# Phase 1 — Schematic + PCB — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (this is EDA
> work, executed by a human in KiCad; steps are checkbox-tracked). Steps use `- [ ]` syntax.

**Goal:** Produce a JLCPCB-ready main board + sensor sub-board + 2 encoder sub-boards
(Gerbers + BOM + CPL) that implement the Phase 0 hardware lock with zero open questions.

**Architecture:** KiCad 8 project. Main PCB-as-chassis carries ESP32-S3, power, motor drivers,
IMU, OLED/rotary headers, and connectors to a front sensor sub-board (16+2 sensors, mux) and
two wheel-axle encoder sub-boards (AS5047P). Design copies the proven IDLine Rev.3 topology
where it overlaps (mux, emitter switch, encoder pull-ups, battery divider) and diverges where
Phase 0 chose different parts (DRV8874 vs BTS7960, OLED vs TFT, AS5047P vs raw quad).

**Tech Stack:** KiCad 8 (free), JLCPCB fab + SMT assembly, LCSC parts. Verification =
ERC clean, DRC clean, netlist spot-checks, and JLC's Gerber/BOM/CPL viewer preview.

## Global Constraints (from Phase 0 — copied verbatim)

- MCU: **ESP32-S3-WROOM-1, non-octal-PSRAM (N8/N16)** so GPIO35–37 are free.
- Pin map is **frozen** — use `docs/hardware/PINMAP.md` net names exactly; no reassignment
  without updating that file first.
- **Sensing nets (`MUX_ADC`, `WING_L`, `WING_R`, `VBAT_SENSE`) are ADC1 (GPIO1–4).** ADC2 banned.
- Rails: `VBAT` (2S), `VMOTOR` (=VBAT, PWM-limited), `VLOGIC_3V3` (MP1584 buck),
  `VSENSE_3V3` (LC/RC-filtered from buck). **Star ground; motor/analog physical separation.**
- Drivers: 2× DRV8874. IMU ICM-42688 on its own I²C. Encoders AS5047P (ABI→PCNT).
- Brownout + watchdog are firmware, but hardware must support clean power (bulk cap, decoupling).
- Reserved (place, do not populate): ESC 3-pin header on `ESC_PWM`=GPIO42 + `VBAT` tap.
- Every SMD part = the LCSC part number in `docs/hardware/BOM.csv`.
- Boards fabbed by JLCPCB with SMT assembly; keep to JLC design rules + assembled-part orientation.

## Reference material

- **IDLine Rev.3 schematic** (proven topology): `D:\LFR\IDLine\Schematics\Rev.3\` and
  `D:\LFR\IDLine\PCB Layout\Rev.3\`. Use for mux wiring, emitter switch, encoder pull-ups,
  battery divider, connector pinouts. **Do not copy their MCU/driver/display — ours differ.**
- Phase 0 docs: `docs/hardware/{BLOCK-DIAGRAM,PINMAP,BOM.csv,POWER-BUDGET}.md`.

## File Structure (KiCad project)

```
hardware/kicad/curly-potato/
  curly-potato.kicad_pro
  main.kicad_sch          # root sheet + hierarchical sheets below
  power.kicad_sch         # buck, reverse-polarity, rails, batt divider
  mcu.kicad_sch           # ESP32-S3, USB-C, boot/reset, decoupling
  motors.kicad_sch        # 2x DRV8874 + motor/encoder connectors
  periph.kicad_sch        # IMU, OLED hdr, rotary hdr, RGB, ESC reserved hdr
  main.kicad_pcb          # main board layout
  sensor/ sensor.kicad_sch sensor.kicad_pcb    # 16+2 sensors + 74HC4067
  encoder/ encoder.kicad_sch encoder.kicad_pcb # AS5047P sub-board (x2, same board)
  fab/                    # Gerbers, BOM.csv (JLC), CPL.csv (JLC) exports
```

---

## Task 1: KiCad project + library setup

**Files:** Create `hardware/kicad/curly-potato/curly-potato.kicad_pro`, `main.kicad_sch`.

- [ ] **Step 1: Create the project.** Install KiCad 8. New project at the path above. Add the
  **JLCPCB/LCSC library** (KiCad plugin "JLC2KiCad" or the community `easyeda2kicad` tool) so
  DRV8874, ICM-42688, AS5047P, MP1584, 74HC4067, ITR8307, ESP32-S3-WROOM-1 come in with the
  correct LCSC footprints + 3D models. Import each MPN from `BOM.csv` by its LCSC number.
- [ ] **Step 2: Verify** every `BOM.csv` JLC-SMD MPN resolved to a symbol+footprint with an
  LCSC field populated. List any that failed to import; find the KiCad-native symbol +
  JLC footprint manually for those. Expected: all parts have footprint + LCSC #.
- [ ] **Step 3: Commit.**
```bash
git add hardware/kicad/curly-potato/
git commit -m "hw(pcb): KiCad project + JLC/LCSC libraries imported"
```

---

## Task 2: Power sheet

**Files:** Create `power.kicad_sch`.

**Interfaces:** Produces nets `VBAT`, `VMOTOR`, `VLOGIC_3V3`, `VSENSE_3V3`, `GND`, `VBAT_SENSE`.

- [ ] **Step 1: Draw the input + protection.** XT30 pad → AO3401 P-FET reverse-polarity
  (source=VBAT_in, gate→GND via 100k + zener clamp, drain=`VBAT`) → bulk input cap. Tap
  `VMOTOR` = `VBAT` directly (motor rail). Add a power switch pad/header.
- [ ] **Step 2: Buck (MP1584EN) → `VLOGIC_3V3`.** Follow MP1584 datasheet ref design: 22 µH
  inductor, input/output caps, feedback divider set for 3.3 V. Add enable pull-up.
- [ ] **Step 3: `VSENSE_3V3` filter.** From `VLOGIC_3V3` through a ferrite bead (or 10 Ω) +
  10–47 µF → `VSENSE_3V3` (clean analog/emitter rail). Do NOT feed sensors from the raw buck node.
- [ ] **Step 4: Battery divider.** `VBAT` → 10k/3k divider → `VBAT_SENSE` (matches IDLine),
  with a small cap to GND; confirm divided max (8.4 V·3/13 = 1.94 V) is within ADC1 range.
- [ ] **Step 5: Verify — run ERC on this sheet.** Expected: no unconnected-power, no
  driving-conflict errors. Confirm divider math note on schematic.
- [ ] **Step 6: Commit.** `git commit -m "hw(pcb): power sheet — buck, rails, protection, batt divider"`

---

## Task 3: MCU sheet

**Files:** Create `mcu.kicad_sch`.

**Interfaces:** Consumes `VLOGIC_3V3`, `GND`. Produces every GPIO net named in `PINMAP.md`.

- [ ] **Step 1: Place ESP32-S3-WROOM-1.** Wire `3V3`, `GND` (all pads), `EN` with RC reset +
  reset button, `GPIO0` boot button. Decouple with 0.1 µF per power pad + 10 µF bulk.
- [ ] **Step 2: USB-C.** D+/D- → GPIO19/20 (native USB), CC pull-downs, VBUS not used for
  power (self-powered from LiPo) — or add VBUS→buck-OR if you want USB power; keep simple: USB
  data + GND only for programming/CDC.
- [ ] **Step 3: Label every functional GPIO net** exactly per `PINMAP.md`
  (`MUX_S0..S3`, `EMIT_EN`, `AIN1/2`,`PWMA`,`BIN1/2`,`PWMB`, `ENCL_A/B`,`ENCR_A/B`,
  `I2C0_SDA/SCL`,`I2C1_SDA/SCL`,`ROT_A/B/SW`, `MUX_ADC`,`WING_L`,`WING_R`,`VBAT_SENSE`,`ESC_PWM`).
- [ ] **Step 4: Verify** — cross-check each schematic net label against the `PINMAP.md` table
  row-by-row; every one of the 27 signals present, on the right GPIO, none duplicated. ERC clean.
- [ ] **Step 5: Commit.** `git commit -m "hw(pcb): MCU sheet — ESP32-S3, USB, reset/boot, frozen pinmap nets"`

---

## Task 4: Motor + encoder sheet

**Files:** Create `motors.kicad_sch`.

**Interfaces:** Consumes `VMOTOR`, motor+encoder GPIO nets, `VLOGIC_3V3`. Produces motor
output pads + encoder sub-board connectors.

- [ ] **Step 1: 2× DRV8874.** Left: `AIN1/AIN2`→IN1/IN2, `PWMA`→ (use IN/IN PWM mode or
  EN/PH — pick PH/EN: `PWMA`→EN, direction→one IN; simplest is IN/IN both PWM-capable — follow
  DRV8874 datasheet Table for PMODE). Add `VM`=`VMOTOR` + 0.1 µF & bulk 470 µF near VM,
  output → motor screw/JST pads, `GND`. Repeat Right with `BIN1/BIN2/PWMB`. Tie PMODE/nSLEEP
  per datasheet (nSLEEP→3V3 via pull-up, optionally GPIO-controlled — if so it needs a pin;
  we have spare GPIO47/48, wire nSLEEP to GPIO47 for a firmware kill-switch).
- [ ] **Step 2: Encoder connectors.** 2× 4-pin JST to the encoder sub-boards: `VLOGIC_3V3`,
  `GND`, A, B → `ENCL_A/B`, `ENCR_A/B`. Add 4k7 pull-ups on A/B (matches IDLine) if the
  AS5047P push-pull output needs none — AS5047P ABI is push-pull, so pull-ups optional; keep
  footprints DNP.
- [ ] **Step 3: Verify** — ERC clean; confirm `nSLEEP` decision reflected back into
  `PINMAP.md` (GPIO47 now used → update spare count to 1). Confirm bulk cap on each `VM`.
- [ ] **Step 4: Commit.** `git commit -m "hw(pcb): motor drivers + encoder connectors"`

---

## Task 5: Peripheral sheet

**Files:** Create `periph.kicad_sch`.

**Interfaces:** Consumes I²C/rotary/ESC nets, `VLOGIC_3V3`.

- [ ] **Step 1: IMU ICM-42688** on `I2C1_SDA/SCL` (its own bus), address-select strap, decouple.
- [ ] **Step 2: OLED header** (4-pin: 3V3,GND,`I2C0_SDA`,`I2C0_SCL`) + I²C pull-ups (4k7) on
  I2C0. **Rotary header** (`ROT_A/B/SW` + 3V3/GND) with RC debounce on SW.
- [ ] **Step 3: Status** WS2812B RGB LED on a spare GPIO (use GPIO48) + a couple of plain LEDs.
  Add the **reserved ESC header** (3-pin: `ESC_PWM`, `VBAT`, GND) — place, mark DNP.
- [ ] **Step 4: Verify** — ERC clean across the whole design (annotate, then ERC). Zero errors,
  warnings triaged. Confirm both I²C buses have pull-ups exactly once.
- [ ] **Step 5: Commit.** `git commit -m "hw(pcb): IMU, OLED/rotary headers, RGB, reserved ESC"`

---

## Task 6: Sensor sub-board

**Files:** Create `sensor/sensor.kicad_sch`, `sensor/sensor.kicad_pcb`.

**Interfaces:** Consumes `VSENSE_3V3`, `GND`, `EMIT_EN`, `MUX_S0..S3`, `MUX_ADC`,
`WING_L`, `WING_R`. Ribbon/board-to-board connector to main board.

- [ ] **Step 1: Schematic.** 16× ITR8307: emitters in series with 100 Ω `R_EMIT` to a common
  `GSENS` node switched by AO3400 (gate=`EMIT_EN`); phototransistors with 10k `R_LOAD` to
  `VSENSE_3V3`, collectors → 74HC4067 inputs. Mux `A/B/C/D`=`MUX_S0..3`, common `X`=`MUX_ADC`.
  2 wing ITR8307 → `WING_L`/`WING_R` direct (own load resistors). Connector carries all nets.
- [ ] **Step 2: PCB.** Lay the 16 sensors in a **straight (or slightly curved) row** across the
  nose at even spacing (~8–10 mm pitch across ~130 mm, tune to your line width — see guide);
  wings at the outer ends, wider. Ground pour both sides. Keep it a thin front strip.
- [ ] **Step 3: Verify** — ERC + DRC clean; confirm sensor pitch vs target line width (line
  must always cover ≥2 sensors for interpolation). Confirm connector pinout matches main board.
- [ ] **Step 4: Commit.** `git commit -m "hw(pcb): sensor sub-board (16+2 ITR8307 + 74HC4067)"`

---

## Task 7: Encoder sub-board

**Files:** Create `encoder/encoder.kicad_sch`, `encoder/encoder.kicad_pcb`.

- [ ] **Step 1: Schematic.** AS5047P with decoupling; ABI outputs A,B → 4-pin connector +
  3V3/GND. Configure for ABI incremental mode (per datasheet OTP/default). Diametric magnet
  sits under the IC.
- [ ] **Step 2: PCB.** Tiny board (~15×15 mm) with a mounting hole so it sits a fixed gap above
  the axle-end magnet. One design, fabbed ×2.
- [ ] **Step 3: Verify** — DRC clean; magnet-to-IC gap note (AS5047P wants ~0.5–2 mm). Confirm
  connector matches the main-board encoder connector from Task 4.
- [ ] **Step 4: Commit.** `git commit -m "hw(pcb): AS5047P encoder sub-board"`

---

## Task 8: Main PCB layout

**Files:** `main.kicad_pcb`.

- [ ] **Step 1: Board outline = chassis.** ~90×100 mm (Phase 0 default). Mount motors at rear,
  wheel cutouts, front edge = sensor-board connector, front skid hole. Place battery over the
  axle (CG low+centered).
- [ ] **Step 2: Placement.** ESP32-S3 central; **keep DRV8874 + `VMOTOR` + bulk cap in a motor
  zone, physically separated from the mux/analog nets**; buck near battery input; OLED/rotary
  at the rear top for driver access; USB-C at rear edge.
- [ ] **Step 3: Route power first** (wide `VMOTOR`/`GND`), then analog (`MUX_ADC` short, guarded),
  then digital. **Star ground**: motor-return and logic-return join at one point by the battery.
  No analog trace under/beside a motor/PWM trace. Copper pour GND both layers, stitched.
- [ ] **Step 4: Verify — DRC clean** at JLC rules (min trace/space, via, annular). Run the
  **JLCPCB DRC profile**. Confirm all assembled parts have valid footprints + courtyards clear.
  Zero DRC errors.
- [ ] **Step 5: Commit.** `git commit -m "hw(pcb): main board layout (star ground, motor/analog separation)"`

---

## Task 9: Fab outputs + JLC pre-flight

**Files:** Create `hardware/kicad/curly-potato/fab/` (Gerbers, `BOM.csv`, `CPL.csv`).

- [ ] **Step 1: Export** Gerbers + drill (JLC preset), the **JLC BOM** (Comment/Designator/
  Footprint/LCSC) and **CPL/position** file for each board (main, sensor, encoder).
- [ ] **Step 2: Pre-flight in JLC.** Upload Gerbers to JLCPCB Gerber viewer; upload BOM+CPL to
  the assembly step; confirm every SMD part **matches a JLC part with live stock** (this closes
  the Phase 0 `verify@order` gate), and part **rotations/polarity** preview correctly.
- [ ] **Step 3: Verify — order-readiness checklist** in a short `fab/READY.md`: Gerbers pass
  viewer, all assembly parts in stock, no unrotated polarised parts, board dims correct, total
  quoted cost ≤ budget. Any red item blocks ordering.
- [ ] **Step 4: Commit + push.**
```bash
git add hardware/kicad/curly-potato/fab/
git commit -m "hw(pcb): fab outputs (Gerber/BOM/CPL) + JLC pre-flight ready"
git push
```

---

## Self-Review (plan author)

- **Coverage:** power (T2), MCU+USB (T3), drivers+encoders (T4), IMU/UI/reserved (T5), sensor
  board (T6), encoder board (T7), layout with noise discipline (T8), fab + stock-gate close (T9).
  All Phase-0 subsystems mapped. Deviation captured: `nSLEEP`→GPIO47 consumes one spare pin
  (update PINMAP in T4).
- **No placeholders:** each task cites exact sheet, nets, and the ERC/DRC verification gate.
- **Consistency:** every net name traces to `PINMAP.md`; rail names match `BLOCK-DIAGRAM.md`.

## Next
Phase 2 — firmware skeleton + HAL + native unit tests (control math is unit-testable
off-hardware; this is where TDD/pytest-style granularity fully applies). Reuse the IDLine
patterns catalogued in `docs/reference/IDLINE-RE-INSIGHTS.md`.
