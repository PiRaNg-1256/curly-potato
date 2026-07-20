# Fast Line Follower Robot — Design Spec

**Date:** 2026-07-20
**Status:** Approved design → ready for implementation planning
**Codename:** working title "LFR-S3"

---

## 1. Goal & Scope

Build a **top-tier, extremely fast line-following race robot** whose defining strengths are **raw straight-line speed** and **control through sharp, high-speed turns**.

### In scope (v1)
- Follow a line that may be **black-on-white or white-on-black**, switchable at runtime.
- Traverse arbitrary line **patterns**: gentle curves, sharp 90° corners, T-junctions, crosses, and short **line gaps**.
- On-robot **screen + input** to pick modes (calibrate, polarity, speed profile, learn/race, telemetry).
- **Wireless live tuning** (WiFi/BLE) so PID and speed params change without reflashing.
- **Encoders + IMU designed in from day one** (foundation for track mapping).
- **Suction/downforce fan** supported as an *optional bolt-on module* — fitted only if traction-limited at target speeds. Not part of v1 base mass.

### Explicitly out of scope (v1)
- **Obstacle avoidance** — dropped.
- **Uneven terrain / seesaws** — dropped.
- **Inverted / upside-down running** — dropped for now; hardware leaves a path to add it later (see §12).

### Success criteria
- Completes a standard competition line course with sharp turns without derailing at the chosen speed profile.
- "Insane" speed profile is meaningfully faster than a plain reactive-PID robot on the same track.
- No **runaway** on line loss (controlled stop) and no random resets/glitches under motor load (the user's stated top concern).

---

## 2. Control Architecture

**Chosen path: Architecture B for v1, with Architecture C (track mapping) as a later selectable mode.**

- **B — Reactive PID + adaptive speed + sensor fusion.** Sensor array → interpolated line position → PID steering; base speed modulated by curvature (slow into corners, launch on straights); **wheel encoders** give closed-loop wheel-speed control; **IMU gyro** gives yaw rate for turn detection and gap traversal. This alone exceeds every reference repo on hand.
- **C — Track mapping (future mode).** Lap 1 records the track as a sequence of segments (straight lengths + turn angles) from encoders + gyro. Lap 2+ **replays** the profile — braking before known corners, flooring known straights — with the sensor array only correcting drift. This is the real "how the fastest bots win" unlock. Encoders + IMU are specified now precisely so C drops in without a hardware change.

Rationale for B-first: get a robust, fast, well-tuned reactive robot on the track quickly; C is high value but riskier and benefits from a proven B baseline underneath it.

---

## 3. Microcontroller

**ESP32-S3 (ESP32-S3-WROOM-1 module).**

- 240 MHz dual-core Xtensa + FPU — ample for a 16-sensor read + fused control loop at 1–2 kHz.
- **Built-in WiFi/BLE → wireless live tuning** (biggest dev-time saver; removes the reflash-retest loop).
- JLCPCB-assemblable module; large community; proven at competition level on the IDLine reference hardware.
- Dual-core split: **Core 1** runs the hard-real-time control loop; **Core 0** runs UI, telemetry, and wireless.

**Migration note:** if raw compute ever becomes the bottleneck (unlikely for this design), the firmware's hardware-abstraction layer (§8) is written so a port to **Teensy 4.1 (600 MHz M7)** touches only the HAL, not the control logic.

---

## 4. Sensing

### 4.1 Line array
- **15–16 analog reflectance channels** (IR emitter LED + phototransistor pairs), spaced across a wide front nose.
- Read through a **74HC4067 16:1 analog mux → one fast ADC channel** (deterministic, cheap, low pin count). Alternative kept in reserve: a dedicated SPI ADC if mux-settling limits loop rate.
- **Analog, not digital** — enables sub-sensor **interpolated position** (weighted centroid) → smooth, high-resolution error instead of quantized steps. This is central to smooth high-speed control.
- Array is a **separate sensor PCB** on the front nose, connected by a short ribbon/board-to-board connector so caster/geometry can be tuned without respinning the main board.

### 4.2 Wing sensors
- **2 outrigger sensors** (one per side, wider than the main array) to detect **90° corners, T-junctions, crosses, and the start/finish marker**. Lets the robot *anticipate* a hard turn instead of reacting late.

### 4.3 Odometry & attitude
- **2 wheel encoders** (magnetic quadrature on the motor shaft or wheel) — closed-loop wheel speed + distance (needed for C).
- **1 IMU** (6-axis gyro+accel, e.g. MPU-6500/ICM-class over SPI) — yaw rate for turn detection, gap traversal, and mapping.

### 4.4 Invertible black/white
Handled entirely in firmware: calibration captures min/max per channel; a **polarity flag** (set from the menu) decides whether "line = high reflectance" or "line = low reflectance." No hardware change.

---

## 5. Motors, Driver, Drivetrain

- **Motors:** high-RPM small brushed **coreless** motors (or low-gear-ratio micro-metal gearmotors chosen for high output RPM). Two, differential drive.
- **Wheels:** small-diameter with **grippy silicone tires** — traction is the cornering-speed limiter; this is where a suction fan earns its place only if grip runs out.
- **Driver:** **per-motor H-bridge drivers (DRV8874-class, ~2 A+ continuous, high PWM frequency, low R_DS(on))** for punchy response and headroom over coreless current spikes. High PWM frequency (≥ 20 kHz) keeps motors quiet and control smooth.
- **Differential mixing** with the option of **inside-wheel reverse (pivot)** on the sharpest turns.

---

## 6. Power

- **2S LiPo** primary pack.
- **Motor rail:** LiPo direct, speed governed by PWM (keeps torque high).
- **Logic rail:** **buck converter to 3.3 V** for MCU + digital.
- **Analog/sensor rail:** separately filtered/regulated, kept away from motor noise.
- **Noise discipline (this is the anti-"random-error" hardware story):** bulk + local decoupling caps, motor kickback protection, **star ground**, motor traces routed away from analog sensor traces, ferrite/RC on noisy lines, brownout detection enabled. Motor electrical noise corrupting the analog array or resetting the MCU is the #1 cause of "random errors" in fast LFRs — the PCB is designed against it deliberately.

---

## 7. Chassis & PCB

- **The main PCB is the chassis** (standard for fast LFRs) — no separate frame.
- **Geometry:** low, wide, **short wheelbase** for agility; center of gravity **low and centered over the drive axle**; front of board carries the sensor-array nose (or ribbon-linked sensor PCB); a **low-friction front skid/ball** as the third contact.
- **Suction provision:** a central mounting zone + an **ESC control header and power path** on the PCB so a brushless **EDF downforce fan + skirt** can be added later without a respin.
- **Fabrication:** JLCPCB SMT-assembled. Parts chosen from JLC's stocked catalog where possible; the ESP32-S3 module, connectors, and any through-hole (encoders, LiPo leads, headers) are placed for straightforward assembly.

---

## 8. Firmware Architecture

Layered, with a hard-real-time core isolated from everything slow.

```
┌── Core 1 (real-time, hardware-timer ISR, fixed rate 1–2 kHz) ──┐
│  read_sensors() → filter → position()                          │
│  fuse(encoders, imu)                                           │
│  control_step()  (PID steering + adaptive speed)               │
│  motor_mix() → drive()                                         │
│  safety_check() (line-loss, brownout, watchdog kick)           │
└────────────────────────────────────────────────────────────────┘
┌── Core 0 (soft real-time) ─────────────────────────────────────┐
│  UI/menu + screen   |  wireless tuning/telemetry  |  NVS config │
└────────────────────────────────────────────────────────────────┘

HAL (pins, ADC/mux, PWM, encoder capture, IMU bus)  ← only layer
                                                       that is MCU-specific
```

**Discipline rules (all chosen to kill random errors):**
- **Fixed-rate control loop via hardware timer / dedicated core** — never `delay()`, never blocking I/O in the loop.
- **No dynamic allocation** anywhere in the loop; fixed-size arrays only.
- **Median/rolling filter** on ADC to reject glitches; **hysteresis** on line-present detection.
- **Watchdog timer** kicks each loop; a hang → safe reset.
- **Brownout detector** enabled.
- **NVS (flash) config persistence** — calibration, polarity, selected profile, PID gains survive power cycles.
- **Power-on self-test** — verify sensors respond, motors wired, IMU present, battery voltage sane; refuse to run if a check fails.
- **Debounced** menu input.

---

## 9. Control Logic (the "make it fast" detail)

Per control tick (pseudocode, not final code):

```
raw[0..15]   = read_mux_adc()
cal[i]       = normalize(raw[i], min[i], max[i])           // calibration
if polarity == WHITE_LINE: cal[i] = 1 - cal[i]             // invertible B/W

position     = weighted_centroid(cal[])   // interpolated, sub-sensor resolution
error        = position - center
line_present = any(cal[i] > present_threshold)  // with hysteresis

// ---- steering ----
d_error      = error - prev_error            // (rate; Kd dominates sharp turns)
integral    += error                          // clamped (anti-windup)
steer        = Kp*error + Ki*integral + Kd*d_error

// ---- adaptive / feed-forward speed ----
curvature    = f(|error|, |d_error|, imu.yaw_rate, wings)
base_speed   = profile.max_speed * (1 - k_curve * curvature)   // slow into curves
base_speed   = launch_boost(base_speed, straight_run)          // floor straights

// ---- closed-loop wheel speed (encoders) ----
left_target  = base_speed - steer
right_target = base_speed + steer
left_pwm     = wheel_pid(left_target, encoder_left_speed)
right_pwm    = wheel_pid(right_target, encoder_right_speed)
allow inside-wheel reverse when |steer| very large (pivot)

// ---- corner / junction anticipation ----
if wing detects 90°/T/cross: pre-brake + commit turn direction
// ---- line loss ----
if !line_present for N ticks: search using last error sign; then controlled stop
```

**Why this is fast, concretely:**
1. **Interpolated position** → smooth error → less oscillation → can run faster before it wobbles.
2. **Kd-dominant PID** → crisp correction into sharp turns without overshoot.
3. **Adaptive speed** → brakes *before* corners, floors straights, instead of one flat speed.
4. **Encoder closed-loop** → both wheels actually hit target speed (battery sag / motor mismatch no longer cause drift).
5. **Wing anticipation + IMU** → the robot *knows* a hard turn is coming and sets up for it.
6. **Track mapping (mode C, later)** → converts "react" into "plan": brake points and launch points learned per-track.

---

## 10. Screen Modes / UI

Screen: **SSD1306 OLED (I²C)**. Input: **rotary encoder with push** (or 3 tactile buttons). Menu tree:

1. **Calibrate** — sweep robot over line+background; capture per-channel min/max.
2. **Line polarity** — Black-on-White / White-on-Black.
3. **Speed profile** — Safe / Fast / Insane (each bundles base speed, PID gains, accel caps).
4. **Drive mode** — Reactive (B) / Track-Learn / Race (C, when built).
5. **Suction** — On / Off (only if fan module fitted).
6. **Telemetry / Live tune** — show error, speeds, battery; adjust gains (also over wireless).
7. **Self-test** — sensors, motors, IMU, battery.

All selections persist to NVS.

---

## 11. Testing & Bring-up Plan

Staged, so faults are caught in isolation (prevents chasing "random" ghosts):

1. **Bench electrical:** power rails, brownout, current draw; no firmware.
2. **Sensor bring-up:** raw ADC per channel, calibration, interpolated position on a static line.
3. **Motor/encoder bring-up:** open-loop drive, then closed-loop wheel-speed PID verified against encoders.
4. **IMU bring-up:** yaw-rate sanity, turn detection.
5. **Reactive follow (B), slow:** tune Kp/Kd on gentle track; verify no oscillation.
6. **Adaptive speed + wings:** add corner anticipation; step speed profiles up.
7. **Robustness soak:** long runs under full motor load watching for resets/glitches; validate watchdog + line-loss stop.
8. **(Later) Track mapping (C):** learn lap, replay, tune brake/launch points.
9. **(Conditional) Suction:** fit fan only if step 6/7 shows traction-limited slide-out in corners.

---

## 12. Future Hooks (designed-in, not built in v1)

- **Track mapping (C)** — encoders + IMU already present; add the record/replay layer.
- **Suction/downforce** — ESC header + power path + mount already on the PCB.
- **Inverted running** — same suction path; would add hardware to survive/steer while inverted.
- **Teensy 4.1 port** — isolated behind the HAL.

---

## 13. Key Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Motor noise → sensor glitches / MCU resets ("random errors") | Star ground, rail separation, decoupling, brownout+watchdog, median filter (§6, §8) |
| Traction limit → slide out in fast corners | Grippy silicone tires first; suction fan module as the fallback (§5, §7) |
| Mux settling limits loop rate | SPI ADC alternative held in reserve (§4.1) |
| Over-scope / never-ships | Phased plan; B ships before C; suction/inverted deferred (§1, §11) |
| Tuning pain | Wireless live tuning from day one (§3, §10) |
| JLC part availability | Prefer JLC-stocked parts; keep footprints for alternates |

---

## 14. Phased Roadmap (for planning)

- **Phase 0 — Architecture & BOM lock:** finalize exact parts (motor P/N, driver P/N, IMU P/N, encoder type, connectors), pin map, power budget.
- **Phase 1 — Schematic + PCB (main + sensor board):** design, DRC, JLC assembly export.
- **Phase 2 — Firmware skeleton + HAL:** boot, timer loop, self-test, NVS, menu, wireless link.
- **Phase 3 — Sensor + motor + encoder + IMU bring-up.**
- **Phase 4 — Reactive follow (B) + tuning.**
- **Phase 5 — Adaptive speed + wing anticipation + speed profiles.**
- **Phase 6 — Robustness soak + fail-safes.**
- **Phase 7 (later) — Track mapping mode (C).**
- **Phase 8 (conditional) — Suction module.**

Each phase gets its own detailed implementation plan.
