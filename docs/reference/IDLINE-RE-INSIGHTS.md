# IDLine Reverse-Engineering — Insights Adopted

Source: `D:\IDLine\reverse-engineered\` (decompiled from IDLine v1.3.5 firmware in a prior
session). IDLine is a competition-grade ESP32 line follower. This file records what we
**adopt** into curly-potato and what we **skip**. Treat the RE as functionally-equivalent
reconstruction, not original source — verify exact constants against `decompiled/` before
copying any literal.

## Hardware topology — CONFIRMS our locked design

| IDLine (proven) | curly-potato (locked) | Match? |
|---|---|---|
| 16 sensors via 74HC4067 mux, select A/B/C/D, common analog X | same | ✅ |
| Emitter LEDs switched by transistor (`ESENS`→Q1→`GSENS`) | `EMIT_EN`→AO3400, strobed | ✅ |
| Quadrature encoders, 4k7 pull-ups | AS5047P ABI (axle) → PCNT | ✅ (ours magnetic) |
| IMU on I²C ("Wire") bus | ICM-42688 on own I²C | ✅ |
| Battery sense via 10k/3k divider | VBAT_SENSE divider (ADC1) | ✅ |
| Motor PWM 8-bit, freq clamp [300,10000] Hz | LEDC PWM ≥20 kHz | ✅ (ours higher) |
| Display + resistor-ladder buttons / rotary | OLED + rotary encoder | ✅ (ours simpler) |

IDLine drives motors with **BTS7960** (43 A half-bridges) — bigger than our DRV8874 because
IDLine also lifts a gripper + runs a vacuum fan. For a pure coreless line-racer, DRV8874 is
right-sized; revisit only if the chosen motor's stall current exceeds ~2 A.

## Firmware patterns — ADOPT

1. **Calibration model** (exact): per sensor, per profile store `low`(min) and `high`(max);
   `threshold = high − (high−low)·sensitivity/100`. 5 calibration profiles for different
   track/lighting. → use verbatim in Phase 2 sensor code.
2. **Multiple PID profiles** (they ship 5: Kp, Ki, Kd, maxSpeed, minSpeed, sampleMs — param-major).
   Kp & Ki stored ×100 (divide by 100.0 at use), Kd direct. → our Safe/Fast/Insane become
   named entries in the same structure.
3. **Line-color per context** (BLACK LINE / WHITE LINE) — invertible, even per plan-segment.
   → confirms our runtime polarity flag; extend it to be per-segment in mode C.
4. **Config persistence via LittleFS** files (`/sensor.txt`, `/PID.txt`, `/GPIO.bin`,
   `/defaultPlan.txt`). → **switch our config store from NVS to LittleFS** — enables the
   web UI to upload/download plans as files, and larger structured data than NVS suits.
5. **Web UI over WebSocket** (`/ws`, `CMD:SAVE`, `PLAN:<csv>`, manual-drive verbs). → our
   wireless tuning grows into a small web app in a later phase; protocol shape is a template.

## Mode C (track memory) — REDESIGNED around IDLine's model

IDLine does NOT use raw encoder odometry to "map" the track. It uses a **checkpoint-indexed
path plan**:

- The track carries **side-marks** (checkpoints). A dedicated scan sensor counts them.
- A **plan** is an ordered list of **steps**; each step declares: which sensors to follow,
  line color, base speed, PID profile, how many checkpoints to count before acting, and an
  action (turn/stop/etc.) with acceleration/deceleration ramps.
- At each counted checkpoint the robot advances to the next step → different speed/PID for
  each track segment. Straights get `maxSpeed` + aggressive profile; known corners get
  pre-set lower speed + damped profile. **This is the "how it's so fast."**

**Our mode C = this, plus our extras:** wing sensors are the checkpoint counters; encoders +
IMU refine *where within a segment* we are (distance/heading) so braking points are precise
even between checkpoints. Best of both: robust checkpoint segmentation + odometry precision.

Our `PlanStep` will be a trimmed version of IDLine's 22-field record (drop gripper/lift/fan
fields we don't use): index, selSensor, scanSensor, colorPlan, speedPlan, pidPlan, followPlan,
countScan, action, timerAction, speedMotorM/P, acc/decTimer, pidTimer.

## SKIP (IDLine features out of our scope)

Gripper servos, lift servo, vacuum fan choreography, robot-to-robot "SHARE PLAN", per-robot
ID activation gate. (Fan header still reserved in hardware per spec, just no choreography.)

## Where to look in the RE when implementing

- Sensor scan + mux: `src/idlinebot_hal.cpp`, `decompiled/FUN_420043f4.c`
- Calibration + polarity auto-detect: `src/idlinebot_line.cpp`
- Line position + PID: `src/idlinebot_control.cpp` (`FUN_42005204`, `FUN_42006068`)
- Plan interpreter (mode C reference): `src/idlinebot_plan.cpp`, `decompiled/FUN_420242be.c`
- Config structs: `src/idlinebot_config.h`
- Web protocol: `src/idlinebot_web.cpp`, `web-ui/page1_*.html`
