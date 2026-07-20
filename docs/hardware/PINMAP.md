# ESP32-S3 Pin Map

Single source of truth for GPIO assignment. Consumed by Phase 1 schematic and every
firmware phase. Signal names here are used verbatim in the HAL.

## Module variant dependency (IMPORTANT)

Assumes **ESP32-S3-WROOM-1 with QUAD flash and NO octal PSRAM** (e.g. `-N8` or `-N16`).
On these, **GPIO35/36/37 are free** (they are the octal-PSRAM pins, unused here).
If an **octal-PSRAM (`R8`) variant** is ever substituted, GPIO35–37 are consumed
internally — the 4 signals on 35/36/38 must move to the spare pins (GPIO47/48) and
the reserved ESC pin must be dropped. **Lock a non-R8 variant in BOM.**

## Hard assignment rules (ESP32-S3)

1. **Sensor mux output + battery sense MUST be ADC1** (GPIO1–GPIO10). **ADC2 is banned**
   for sensing — it is unusable while WiFi is on, and wireless tuning needs WiFi.
2. **GPIO19/20 = native USB D-/D+** → reserved for USB CDC programming/telemetry. Never reuse.
3. **GPIO43/44 = UART0 TX/RX** → left free for serial console/logging.
4. **Strapping pins GPIO0/45/46** → avoided for functional signals. GPIO3 is a strapping
   pin (JTAG source select) but is used only as a passive **ADC input** (safe: input, boot
   default is fine).
5. Encoders use the **PCNT** hardware pulse-counter peripheral; motor PWM uses **LEDC**;
   both map to any GPIO.

## Pin table

| Signal | GPIO | Peripheral | Dir | Notes |
|--------|------|-----------|-----|-------|
| `MUX_ADC`  | 1  | ADC1_CH0 | in (analog) | 74HC4067 common output (16 line sensors) |
| `WING_L`   | 2  | ADC1_CH1 | in (analog) | Left wing sensor, read direct (no mux latency) |
| `WING_R`   | 3  | ADC1_CH2 | in (analog) | Right wing sensor. GPIO3 = strap pin, input-only use OK |
| `VBAT_SENSE`| 4 | ADC1_CH3 | in (analog) | Battery divider tap |
| `MUX_S0`   | 5  | GPIO | out | 74HC4067 select bit 0 |
| `MUX_S1`   | 6  | GPIO | out | select bit 1 |
| `MUX_S2`   | 7  | GPIO | out | select bit 2 |
| `MUX_S3`   | 8  | GPIO | out | select bit 3 |
| `EMIT_EN`  | 9  | GPIO | out | IR emitter enable/strobe (power saving + ambient subtraction) |
| `AIN1`     | 10 | GPIO | out | Left driver IN1 (DRV8874) |
| `AIN2`     | 11 | GPIO | out | Left driver IN2 |
| `PWMA`     | 12 | LEDC | out | Left driver PWM (≥20 kHz) |
| `BIN1`     | 13 | GPIO | out | Right driver IN1 |
| `BIN2`     | 14 | GPIO | out | Right driver IN2 |
| `PWMB`     | 21 | LEDC | out | Right driver PWM |
| `ENCL_A`   | 15 | PCNT | in | Left encoder A |
| `ENCL_B`   | 16 | PCNT | in | Left encoder B |
| `ENCR_A`   | 17 | PCNT | in | Right encoder A |
| `ENCR_B`   | 18 | PCNT | in | Right encoder B |
| `I2C0_SDA` | 35 | I2C0 | io | OLED bus (SSD1306) |
| `I2C0_SCL` | 36 | I2C0 | out | OLED bus |
| `I2C1_SDA` | 37 | I2C1 | io | IMU bus (own controller — no OLED contention) |
| `I2C1_SCL` | 38 | I2C1 | out | IMU bus |
| `ROT_A`    | 39 | GPIO/PCNT | in | Rotary encoder A |
| `ROT_B`    | 40 | GPIO/PCNT | in | Rotary encoder B |
| `ROT_SW`   | 41 | GPIO | in | Rotary push button (debounced) |
| `ESC_PWM`  | 42 | LEDC | out | **Reserved** — future suction fan ESC (not populated v1) |

## Conflict + fit check (Step 2 verification)

- **No GPIO used twice** — every row above has a unique GPIO. ✅
- **ADC1-only sensing** — `MUX_ADC`, `WING_L`, `WING_R`, `VBAT_SENSE` all on GPIO1–4 (ADC1). No ADC2 used. ✅
- **USB/UART/strap respected** — 19/20 (USB), 43/44 (UART), 0/45/46 (strap) all unused by functional signals; GPIO3 used input-only. ✅
- **Two I²C controllers used** (I2C0 OLED, I2C1 IMU) — no shared-bus stall. ✅
- **Signal count:** 27 assigned (4 analog + 23 digital).

## GPIO budget

- Assigned: **27**
- **Free spare (clean, usable now): GPIO47, GPIO48** (2) — reserved for layout bodges / a 3rd wing / a start-line beam-break.
- Reserved-by-function (not counted as free): 19,20 (USB), 43,44 (UART console).
- Avoid: 0,45,46 (strapping).
