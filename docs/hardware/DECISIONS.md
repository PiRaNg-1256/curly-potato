# Hardware Decision Log (ADR-style)

Each entry: choice + one-line why + rejected alternative. Newest last.

## D1 — MCU: ESP32-S3-WROOM-1 (non-octal-PSRAM, N8/N16)
Wireless live tuning + enough compute for 1–2 kHz fused loop; non-R8 so GPIO35–37 free.
Rejected: Teensy 4.1 (no wireless, module mount) — kept as HAL-isolated migration path only.

## D2 — Sensing: 16 analog ITR8307 via 74HC4067 mux + 2 direct wings
Analog → sub-sensor interpolation → smooth high-speed error. Single mux = one ADC net.
Wings read direct on ADC1 for zero-latency corner anticipation.
Rejected: digital QTR modules (no interpolation); all-through-mux (wings want low latency).

## D3 — IMU on its own I²C bus (not SPI)
Saves 2 GPIO vs SPI and, on a dedicated I²C controller, a slow OLED write cannot stall a
gyro read. ICM-42688-P supports I²C ≤1 MHz — ample for ≤2 kHz sampling.
Rejected: SPI IMU (4 pins, near-zero pin margin); shared-bus I²C with OLED (contention).

## D4 — Drivers: 2× DRV8874 (per-motor)
2.1 A continuous + current sense + fast switching → headroom over coreless current spikes.
Rejected: TB6612FNG (marginal current for coreless), L298N (huge Vdrop, slow — the "slow
reference repo" driver).

## D5 — Motors: import premium coreless (user choice)
Highest RPM ceiling for raw straight-line speed. Sourced AliExpress (2S-capable).
Rejected: N20 metal-gear (lower RPM ceiling). Consequence → D6.

## D6 — Encoders: AS5047P magnetic ABI on wheel axle (diametric magnet)
Coreless motors have no rear shaft, but Architecture B/mode C need encoders day 1.
AS5047P outputs ABI quadrature → the PCNT pins already in PINMAP; magnet mounts on the
wheel axle end via a small encoder sub-PCB.
Rejected: AS5600 (I²C absolute — both I²C buses already used by OLED/IMU, fixed address
blocks 2 units); skipping encoders (kills closed-loop wheel speed + track mapping).
**Open at gate:** magnet-on-axle mounting adds mechanical complexity — validate in Phase 1.

## D7 — Power: 2S LiPo, MP1584 buck to 3.3 V, split VLOGIC/VSENSE, reverse-polarity FET
2S gives coreless headroom; MP1584 3A covers logic+sensor; separate filtered VSENSE keeps
motor noise off the analog array (the "random errors" defense). AO3401 P-FET protects VBAT.
Rejected: linear 3.3 V reg (heat/inefficiency at 2S), single unfiltered 3V3 rail (noise).

## Sourcing model
- **JLC-SMD** rows → designed into PCB, JLCPCB SMT-assembled. Authoritative stock/price
  confirmed as ONE JLCPCB cart check at the Phase 0 gate (Task 7) — live per-part scraping
  is unreliable and stock shifts daily; `verify@order` marks parts to confirm in that cart.
- **IMPORT-MECH** rows → motors/wheels/LiPo/OLED/rotary/hardware from AliExpress (import OK).
- **RESERVED** → footprint/header placed, not populated (suction fan, future).

## D8 — Adopt IDLine RE patterns (config store + mode-C model)
From decompiled IDLine (`docs/reference/IDLINE-RE-INSIGHTS.md`):
- **Config/plan persistence: LittleFS files** (not NVS) — supersedes spec §8 NVS line.
  Enables web upload/download of plans and larger structured data. NVS may still hold a
  tiny boot-critical subset.
- **Mode C = checkpoint-indexed path plan** (wing sensors count side-marks → per-segment
  speed/PID/color/action), refined by encoder distance + IMU heading — supersedes the
  "pure encoder odometry mapping" framing in spec §2/§9.
- Reuse: 5 PID profiles + 5 calibration profiles; `threshold = high−(high−low)·sensi/100`;
  web-over-WebSocket tuning.
Rejected: NVS-only (can't serve file-based web plan editing); pure-odometry mapping
(less robust than checkpoint segmentation).

## Cost roll-up (one robot)

| Bucket | Est. USD |
|---|---|
| JLC-SMD parts | ~$23 |
| Import-mechanical (motors/wheels/LiPo/OLED/rotary/hw) | ~$71 |
| PCB fab + SMT assembly (main + sensor + 2 encoder sub-PCBs, small qty) | ~$60 |
| Subtotal | ~$154 |
| +20% spares | ~$31 |
| +shipping / import buffer | ~$40 |
| **Total** | **~$225** |

Budget ceiling $400 → **PASS**, ~$175 headroom (available to upgrade motors/tires if the
first coreless choice underperforms).

## Phase 0 Gate

- [x] Block diagram + 4 rails defined, subsystem-complete — `BLOCK-DIAGRAM.md`
- [x] Pin map: 27 signals, 0 conflicts, sensing ADC1-only, 2 spare GPIO — `PINMAP.md`
- [x] BOM: every part has a concrete MPN; JLC-SMD rows carry LCSC #/`verify@order`;
      sourcing split (JLC vs import) explicit — `BOM.csv`
- [x] Power budget: buck ≥3.1× headroom, LiPo peak ≥ motor stall, battery-life estimated,
      R_EMIT/bulk-cap/inductor locked — `POWER-BUDGET.md`
- [x] Cost ≤ budget — this section
- [ ] **User gate review** — confirm motor listing, AS5047P-on-axle mounting (D6 open item),
      and run one JLCPCB cart check to confirm live stock/price of `verify@order` parts

Phase 0 is design-complete pending the user gate review above. Once confirmed → Phase 1
(schematic + PCB) can start with no open hardware questions.

