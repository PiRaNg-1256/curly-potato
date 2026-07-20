# Power Budget

Rails defined in `BLOCK-DIAGRAM.md`. Figures are worst-case with margin. Motor stall/RPM
marked ASSUMED — confirm from the chosen coreless listing and re-check the two flagged lines.

## VLOGIC_3V3 (from MP1584 buck)

| Consumer | Typ | Peak |
|---|---|---|
| ESP32-S3 (WiFi active) | 150 mA | 500 mA (TX bursts) |
| ICM-42688-P IMU | 1 mA | 3 mA |
| SSD1306 OLED | 15 mA | 25 mA |
| AS5047P ×2 | 30 mA | 40 mA |
| misc (LEDs, dividers, buttons) | 20 mA | 40 mA |
| **VLOGIC subtotal** | ~216 mA | **~610 mA** |

## VSENSE_3V3 (filtered from buck; IR emitters + front-end)

- Emitter drive: ITR8307 Vf ≈ 1.2 V, If = 20 mA. `R_EMIT = (3.3 − 1.2)/0.02 = 105 Ω → 100 Ω`.
- 18 emitters × 20 mA = **360 mA if all on continuously**.
- **Strobed** via `EMIT_EN` (on only during the mux sample window) → average draw a fraction
  of that; strobing also enables ambient-light subtraction (read dark, read lit, subtract).
- Phototransistor loads + mux: negligible (<10 mA).
- **VSENSE peak ~370 mA**, average much lower with strobing.

## Buck sizing check

- VLOGIC peak + VSENSE peak ≈ 610 + 370 = **~980 mA** worst-case (emitters un-strobed).
- MP1584EN rated **3 A** → headroom = 3000/980 ≈ **3.1×**. PASS (≥1.5× required).
- Input range 6.0–8.4 V (2S) within MP1584 4.5–28 V. PASS.

## VMOTOR (2S LiPo direct, PWM-limited)

| | per motor | both |
|---|---|---|
| Continuous (ASSUMED coreless) | ~0.8 A | ~1.6 A |
| Stall transient (ASSUMED) | ~2.5 A | ~5.0 A |

- Bulk cap 470 µF low-ESR across VMOTOR absorbs stall/PWM transients near the drivers.
- LiPo peak: 450 mAh × 45C = **~20 A** available ≫ 5 A motor peak + 1 A logic. PASS.
- DRV8874 rated 2.1 A cont / higher peak per motor ≥ assumed draw. PASS (confirm vs real stall A).

## Battery-life estimate

- Average current ≈ motors ~1.5 A (racing duty) + logic ~0.25 A + emitters strobed ~0.1 A ≈ **~1.85 A**.
- 450 mAh / 1.85 A ≈ **~15 min** continuous. Race runs are seconds–minutes → ample; carry spare packs.

## Locked values (feed Phase 1)

- `R_EMIT` = 100 Ω (0603), one per emitter.
- Buck: MP1584EN, 22 µH inductor, 3.3 V out. 3 A rating confirmed sufficient.
- `CAP_BULK` = 470 µF low-ESR across VMOTOR.
- Emitters **strobed** (not always-on) — firmware requirement noted for Phase 2.

## Flagged for confirmation
- Coreless **continuous + stall current** and **no-load RPM** — from the actual purchased
  motor. If stall > ~2 A/motor sustained, revisit DRV8874 vs a higher-current driver.
