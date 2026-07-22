# curly-potato — The Everything Guide

A plain-English walkthrough of the whole project: what we're building, what to buy, how the
code will work, how the PCB happens, and exactly what to do next. Read top-to-bottom once;
after that, use it as a lookup. Jargon is explained in the **Glossary** at the end — anything
in `code font` you don't recognize is defined there.

---

## 0. What we're building, in one paragraph

A small, fast robot that drives itself along a painted line as quickly as possible, including
through sharp corners, and can follow either a black line on white or a white line on black.
The "brain" is an ESP32-S3 microcontroller. It reads a row of light sensors that look down at
the floor, works out where the line is, and spins two wheels at different speeds to steer. The
speed and steering are computed hundreds of times per second by a control algorithm. A tiny
screen + knob lets you pick modes. Later, it can *memorize* a track and run it even faster.

We are building it in **phases**, and right now the design is fully locked and we're about to
order the circuit board.

---

## 1. The big picture — phases (where we are)

Think of it like building a house: plans → foundation → walls → wiring → furniture. Each phase
produces something concrete and is its own detailed plan document.

| Phase | Plain meaning | Status |
|---|---|---|
| **0. BOM/pin/power lock** | Decide every part, every pin, every power number | ✅ DONE |
| **1. Schematic + PCB** | Draw the circuit, lay out the board, send to factory | 📋 planned, next |
| **2. Firmware skeleton** | Basic code: boot, menus, talk to hardware | later |
| **3. Bring-up** | Prove each part works (sensors, motors, encoders, IMU) | later |
| **4. Reactive follow** | It follows a line and you tune it | later |
| **5. Adaptive speed** | It slows for corners, floors straights | later |
| **6. Robustness** | Long runs, no glitches, safe stops | later |
| **7. Track memory (mode C)** | Memorize a track, run it faster | later |
| **8. Suction (optional)** | Add a downforce fan *if* it slides in corners | later |

**You are here:** Phase 0 done. Phase 1 (order the board) is the next action, and it only waits
on you (a) picking motors and (b) doing the JLC cart check (both explained below).

Full docs live in the repo:
- Design spec (the "why"): `docs/superpowers/specs/2026-07-20-fast-line-follower-design.md`
- Hardware lock: `docs/hardware/` (BLOCK-DIAGRAM, PINMAP, BOM.csv, POWER-BUDGET, DECISIONS)
- Phase plans: `docs/superpowers/plans/`
- Lessons from the decompiled IDLine robot: `docs/reference/IDLINE-RE-INSIGHTS.md`

---

## 2. Shopping list — what to buy, with specs

Two buckets. **Bucket A** parts are soldered onto the board *by the JLCPCB factory* — you don't
buy them separately, you just make sure they're in stock (Section 6). **Bucket B** parts you buy
yourself (AliExpress / local) and attach.

### Bucket A — on the PCB (JLC solders these; full list in `docs/hardware/BOM.csv`)
You don't source these individually; they ride along with the board order. Headline parts:
- **ESP32-S3-WROOM-1-N16** — the brain (must be the **N16 or N8**, *not* the `R8` version).
- **74HC4067** — 16-to-1 analog switch for the sensors.
- **18× ITR8307** — the reflective floor sensors (16 line + 2 "wing").
- **2× DRV8874** — motor driver chips.
- **ICM-42688-P** — the motion sensor (gyro).
- **2× AS5047P** — the wheel rotation sensors (encoders).
- **MP1584** — the voltage regulator (makes 3.3 V for the brain).

### Bucket B — you buy and attach

| Item | Exact spec to look for | Qty | ~Price | Notes |
|---|---|---|---|---|
| **Drive motors** | Coreless, 6–7.4 V, **~2000–3000 RPM no-load**, metal gearbox, 3 mm D-shaft. See Section 3 for exact recommendation. | 2 | $15–60 ea | THE speed part — don't cheap out |
| **Wheels + tyres** | ~32–38 mm ⌀, **silicone/soft rubber** (grip!), fits motor shaft/hub | 2 | $8 | grip = corner speed |
| **2S LiPo battery** | 7.4 V, **450–600 mAh, 45C or higher**, with XT30 plug | 1–2 | $10 ea | buy 2 so you can keep racing |
| **Diametric magnets** | NdFeB, ~6 mm ⌀ × 2.5 mm, **diametrically magnetized** (poles across, not through) | 2 | $2 | for the encoders |
| **OLED display** | SSD1306, 0.96", **I²C** (4-pin) | 1 | $3 | the little screen |
| **Rotary encoder** | EC11 type, with push-button | 1 | $1 | the menu knob |
| **Front skid / ball caster** | tiny PTFE skid or 5–8 mm ball caster | 1 | $2 | 3rd contact point |
| **LiPo charger** | 2S balance charger (iMAX B3/B6 or USB 2S) | 1 | $8 | to charge the battery |
| **Misc** | JST wires, standoffs, double-sided tape, ribbon cable | — | $5 | assembly bits |

You will also need, one-time: a **USB-C cable** (to program the board), and optionally a
**LiPo-safe bag** for charging.

---

## 3. Motor recommendation (you asked)

Motors are the single biggest factor in top speed, and you have ~$175 of budget headroom, so
this is the place to spend it. You chose "import premium coreless." Here's the concrete call:

**What matters:** high **RPM** (for top speed), enough **torque** (to accelerate the ~200 g bot
and not bog down in corners), runs on **2S (7.4 V)**, and a **3 mm shaft** to fit wheels/hubs.
With ~32 mm wheels, ~2500 RPM ≈ ~4 m/s top speed — plenty fast; more RPM needs more track.

**Three good options, cheapest to best:**

1. **Value — N20-format coreless, 6–12 V, high-speed geared (~2000–3000 RPM).**
   Search AliExpress "N20 coreless motor high speed 6V" or "N20 coreless 1000-3000rpm". ~$15–20
   each. Totally fine to start; matches our chassis and DRV8874 drivers.

2. **Recommended sweet spot — a quality micro-metal gearmotor in a high-RPM ratio.**
   e.g. Pololu-style 6 V HPCB micro metal gearmotor, low ratio (~10:1) → high RPM + good torque,
   ~$20–25 each. Extremely proven in LFR racing. (Technically "cored," but the performance and
   reliability are excellent and sourcing is easy. If you're set on true coreless, use option 1
   or 3.)

3. **Podium — Faulhaber coreless (1524 / 1717 series), 6 V.**
   The gold standard in winning line followers: best power-to-weight, silky control. ~$40–60
   each (sometimes cheaper used/surplus on AliExpress/eBay). Get these if you want the ceiling
   and can source them.

**My pick for you:** start with **option 1 or 2** (~$40/pair) to get running, and keep the
budget headroom to upgrade to **option 3** later if you want more. All three work with the board
we're designing — motor choice does **not** change the PCB (motors connect by wire).

**What I need back from you before the fab order isn't blocked, but before final tuning:** the
motor's **no-load RPM** and **stall current** (from the listing or datasheet). If stall current
is above ~2 A, tell me — we'd bump the driver. Everything else proceeds without it.

---

## 4. How the robot works (concepts, no code yet)

### 4.1 Seeing the line
Under the front is a row of **16 reflective sensors** (plus 2 "wing" sensors further out). Each
shines infrared light down and measures how much bounces back. **White floor = lots of
reflection; black line = little.** So the sensor(s) over the line read differently from the rest.

Because we read them as **analog** (a range, not just on/off), we can do **interpolation**: if
the line sits between sensor 8 and 9 but closer to 9, we compute a position like "8.7". That
smooth, fine-grained position is what lets the robot run fast without wobbling.

**Calibration** teaches the robot what "white" and "black" read like on *this* track (every
track/lighting differs). We store, per sensor, the brightest (`high`) and darkest (`low`) values,
and a `threshold` in between. (Formula reused from the IDLine robot:
`threshold = high − (high − low) × sensitivity / 100`.)

**Black or white line?** A simple flag flips the logic ("line = dark" vs "line = light"), so the
same robot follows either. It can even switch mid-track.

### 4.2 Deciding how to steer — PID
The robot wants the line centered (position = 8, the middle of 16). The difference between where
the line *is* and center is the **error**. A **PID controller** turns error into a steering
command using three parts:
- **P (proportional):** steer harder the further off-center you are.
- **D (derivative):** react to how *fast* the error is changing — this is what nails sharp
  corners without overshooting.
- **I (integral):** correct slow, persistent drift (used lightly here).

You "tune" three numbers (`Kp`, `Ki`, `Kd`) until it drives cleanly. We'll do this **wirelessly
from your phone** so you're not reflashing code every time — a big time-saver.

### 4.3 Going fast — adaptive speed
A dumb robot uses one speed everywhere and either crawls or flies off corners. Ours **modulates
speed**: on straights (error small and steady) it floors it; approaching a corner (error/steering
rising, or the gyro sensing rotation) it **brakes early**, then powers out. The **wing sensors**
spot 90° corners and junctions *before* the main row does, so it can set up for them.

### 4.4 Knowing the wheels — encoders + gyro
- **Encoders** (AS5047P) measure how far each wheel actually turned. This lets us command a real
  wheel *speed* (closed loop), so a weak battery or a slightly stronger motor doesn't cause drift.
- **Gyro** (ICM-42688) measures turning (yaw). Helps detect corners and bridge short **gaps** in
  the line.

### 4.5 Memorizing a track — mode C (the real speed unlock)
This is how the fastest robots (including the decompiled IDLine) win. The track has small
**side-marks** ("checkpoints"). The robot counts them with its wing sensors and builds/loads a
**plan**: an ordered list of segments, each with its own speed, PID profile, and line color. On
the race run it *already knows* "after checkpoint 3 comes a hard left — brake now," instead of
reacting late. Our version adds encoder distance + gyro heading so braking points are precise
even between checkpoints. We build this **after** the basic follower works well.

### 4.6 Not glitching — the "no random errors" part (your worry)
Fast motors create electrical noise that can corrupt sensor readings or reset the brain — the
classic cause of "random" line-follower misbehavior. We fight it on **two fronts**:
- **Hardware:** separate power rails for motors vs sensors, a filtered sensor supply, a
  "star ground," bulk capacitors, and keeping motor wiring away from sensor wiring on the board.
- **Firmware:** a fixed-rate control loop (no sloppy timing), noise-filtering the sensors, a
  **watchdog** (auto-reset if the code ever hangs), **brownout detection** (handle voltage dips),
  and a rule that losing the line makes it **stop safely, never run away**.

---

## 5. The circuit board (PCB) — what happens in Phase 1

The board is both the electronics *and* the chassis (the robot's body). Steps, plain version:

1. **Schematic** — a diagram of every chip and how they connect. We draw it in **KiCad** (free
   software). Our Phase 1 plan breaks this into sheets: power, brain, motors, extras, sensors.
2. **Board layout** — arrange the parts physically and draw the copper wires. Key rules: keep it
   low and centered for balance, put motor stuff away from sensor stuff (noise), thick wires for
   power.
3. **Sub-boards** — a thin **sensor board** for the nose (so you can adjust its position) and two
   tiny **encoder boards** for the wheels.
4. **Export** — KiCad spits out "Gerber" files (the board artwork) + a parts list (BOM) + a parts
   position list (CPL). These three go to JLCPCB.
5. **Order** — JLCPCB makes the board and solders the Bucket-A chips on. You get finished boards
   in the mail (~1–2 weeks).

You do **not** need to hand-solder the tiny chips — that's the whole point of JLC assembly. You'll
only attach the Bucket-B parts (motors, wheels, screen, battery) with connectors/wire.

The detailed, click-by-click Phase 1 steps are in
`docs/superpowers/plans/2026-07-20-phase1-schematic-pcb.md`. I can also do the KiCad design work
with you when you're ready.

---

## 6. JLCPCB cart / stock check — exact steps (you asked)

This confirms the factory actually has our chips *before* we finalize the board. Do this once:

1. Go to **jlcpcb.com** → make a free account.
2. Top menu → **"Parts"** (this is the LCSC/JLC parts library the assembly uses).
3. For each `verify@order` part in `docs/hardware/BOM.csv`, paste its name (or LCSC number if
   present) into the Parts search box. The three to check first:
   - `ESP32-S3-WROOM-1-N16` (make sure it's the **N16/N8**, not `-N16R8`)
   - `ICM-42688-P`
   - `AS5047P`
   Also worth checking: `DRV8874`, `74HC4067`, `ITR8307`, `MP1584`.
4. For each result, look at:
   - **Stock** number (want it comfortably > the quantity we need).
   - **"Basic" vs "Extended"** label. *Basic* parts are cheapest and always loaded in the
     machine. *Extended* parts add a small one-time fee (~$3) — fine, just note it.
   - Click **"Save to parts library"** / the bookmark, so it's easy to select during assembly.
5. If a part shows **0 stock or is discontinued**, tell me — the BOM lists an `alt_MPN` for the
   important ones (e.g. MPU-6500 for the IMU), and I'll switch it.
6. (Optional, later) The *real* cart happens at the end of Phase 1 when we upload the board files;
   the assembly page will auto-match our BOM to these parts and show a live quote. This early
   check just avoids designing around an out-of-stock chip.

That's it — you're verifying availability, not buying anything yet.

---

## 7. What to do next — your action list

**Right now (unblocks the board order):**
1. **Pick motors** (Section 3). Reply with which option, or the AliExpress listing, and its
   **no-load RPM + stall current** if shown.
2. **Do the JLC stock check** (Section 6) for the 3 key chips. Tell me if any are out of stock.

**Then (I drive, with you):**
3. I execute **Phase 1** — build the KiCad schematic + PCB from the plan. You review the board
   look/shape; I handle the electronics.
4. We export fab files and you place the JLCPCB order (I'll give exact upload steps).
5. While the board ships (~1–2 weeks), I start **Phase 2** firmware so it's ready when boards
   arrive.

**You never have to:** hand-solder fine chips, write the control math yourself, or figure out
pin assignments — those are done or will be done for you. Your jobs are the physical/purchasing
choices and final on-track tuning (which we do together, wirelessly).

---

## 8. Glossary (plain definitions)

- **Microcontroller / ESP32-S3** — the small computer chip that runs the robot's program.
- **Firmware** — the program that runs on the microcontroller.
- **GPIO / pin** — an electrical connection point on the chip; each does one job (read a sensor,
  drive a motor, etc.). "Pin map" = the list of which pin does what.
- **ADC** — the circuit that turns an analog voltage (a sensor's reading) into a number.
- **Analog vs digital** — analog = a smooth range (e.g. 0–4095); digital = just on/off. We read
  sensors as analog to get precise line position.
- **Mux (74HC4067)** — a switch that lets one ADC read 16 sensors one at a time, saving pins.
- **PID** — the steering math (Proportional-Integral-Derivative). Three tunable numbers.
- **Encoder (AS5047P)** — sensor that measures how far a wheel turned.
- **IMU / gyro (ICM-42688)** — sensor that measures the robot's rotation/tilt.
- **PWM** — switching power on/off very fast to control motor speed (like dimming a light).
- **Motor driver (DRV8874)** — chip that takes the tiny control signal and delivers big current
  to the motor.
- **Buck / regulator (MP1584)** — converts battery voltage down to a steady 3.3 V for the chips.
- **2S LiPo** — a rechargeable battery of two cells in series, ~7.4 V. High "C" = can deliver big
  current bursts.
- **Star ground** — wiring trick: all ground returns meet at one point, to keep motor noise out
  of sensors.
- **Watchdog** — a timer that resets the robot if the code ever freezes.
- **Brownout** — a voltage dip; "brownout detection" handles it gracefully.
- **PCB** — printed circuit board (the green board). Ours is also the chassis.
- **KiCad** — free software to design the schematic + PCB.
- **Gerber / BOM / CPL** — the three files a PCB factory needs: board artwork, parts list, parts
  positions.
- **JLCPCB / LCSC** — the factory that makes the board and the parts catalog it solders from.
- **BOM** — Bill of Materials, the parts list.
- **Calibration** — teaching the robot what black/white look like on this track.
- **Checkpoint / path plan (mode C)** — side-marks on the track the robot counts to know where it
  is and change speed per segment.
- **Coreless motor** — a lightweight, fast-responding DC motor type good for racing.
- **Interpolation** — estimating a precise line position between two sensors.
- **HAL** — "hardware abstraction layer," the part of the code that talks to pins, so the rest of
  the code doesn't care which chip it's on (lets us move to a different brain later easily).
```
