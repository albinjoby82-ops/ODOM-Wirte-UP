# Why These Components

## The Core Decision: Sensing on an ESP32-S3, Motion Control on the V5

Every choice in this pod traces back to one split: sensing lives on an ESP32-S3, motion control stays on the V5 brain.

The V5 simply can't host these sensors well. Smart ports run a closed RS-485 protocol, the brain exposes no user I²C/SPI/UART for a BNO085 or TMF8821s, and the ADI 3-wire quadrature path was designed around 360 CPR encoders; it drops counts once you're running 8192 CPR. Even where the brain *can* read a sensor, the read itself is a problem: smart-port devices are polled on the same bus as motor commands, with a 10 ms default and a 5 ms floor, so the data a controller sees is stale and jittery by the time it arrives. On the ESP32, an encoder read is a direct register access, the IMU is a timestamped 100 Hz stream, and dt is known exactly rather than inferred.

That precision needs somewhere to run without interference. Core 0 on the ESP32 handles all I/O; Core 1 runs the math, including 200 Hz MCL prediction, with nothing from the PROS scheduler competing for cycles. That two-core split is the design intent. The first breadboard build runs everything in one task at 100 Hz, which was enough to prove the sensors and the link.

None of this makes the V5 obsolete as a pose source. It was never one. The brain carries no IMU and no tracking encoders of its own. `LinkPoseSource` is its only pose source, and the custom PROS motion code consumes it through `IPoseSource`, which is why `runMotion` gates on `healthy()`, staleness, and `bootId`, because it has to assume the link can go bad.

Motion control itself stays on the V5 regardless, for two reasons that aren't up for debate: motors must be VEX, and you never put a network link inside a feedback loop. The ESP32 is a pose source, full stop. It never touches a motor command.

It's worth being honest about what this buys and doesn't buy. End-to-end latency over the link is roughly at parity with just reading native V5 sensors over the smart port (about 13 ms mean / 26 ms tail in the 460800 baud / 200 Hz analysis). The real advantage isn't speed, it's determinism: synchronous sampling, timestampable delay, and access to sensors the brain physically cannot read. And it's legal: VUR12 explicitly allows external electronics and non-VEX sensors.

## Why the ESP32-S3 Specifically

The S3 has hardware quadrature decoding (PCNT) and two cores that map directly onto the I/O / math split the design needs. It also brings three UARTs (console, BNO085, RS-485), ESP-IDF/FreeRTOS, and comes as a cheap, compact module, with no exotic sourcing. The N8R8 module has 8 MB of PSRAM, but the build leaves it off: the particle filter is only about 19 KB and runs faster in internal SRAM.

The alternatives were rejected for concrete reasons, not vibes:

- **Raspberry Pi (Linux):** scheduler jitter, and no hardware quadrature decoder.
- **AVR Arduino:** interrupt-based counting, software float, too little RAM.
- **Teensy 4.1:** a genuinely better encoder peripheral, but single-core, and a costly rewrite for a problem that isn't currently compute-bound.
- **RT1170:** best on paper, but there's no board suitable for a competition robot.

The ESP32-P4 is the upgrade path if compute ever does become the limiting factor.

Two liabilities come with the choice and need to stay visible: PCNT is 16-bit, so it wraps roughly every 4 revolutions at 8192 CPR, so the overflow ISR has to be high-priority and sanity-checked. And there's no double-precision FPU, so accidental doubles in the math path are a real failure mode, not a style nitpick.

## Encoders: 2× AMT102-V

These give 8192 CPR (2048 PPR × 4), which on a 2.75" wheel works out to roughly 0.027 mm/count, against 0.054 mm for the V5 Rotation Sensor and 0.61 mm for the ADI optical encoder.

But resolution isn't actually the win here. Both smart-sensor options already sit far below the real error floor of the system, so squeezing more counts out of them wouldn't move the needle. The real win is zero transport delay versus smart-port polling, and that win only exists because the encoders go through the ESP32. Wired into ADI ports instead, they'd have to be throttled to 96 PPR, which throws away the entire reason for buying them in the first place.

The trade-off worth naming: these are incremental encoders, with no absolute position across power cycles. That's fine here, because pose is seeded via `SET_POSE` at the start, not recovered from the encoders themselves. Their 5 V outputs go through an SN74LVC245 buffer to bring them down to 3.3 V logic. The physical setup is two-pod diamond odometry, with heading supplied separately by the IMU.

## IMU: BNO085 in UART-RVC Mode

The deciding factor is bias handling under the specific usage pattern of this robot. The BNO085's SH-2 firmware re-estimates gyro bias whenever the robot is stationary, and this robot stops every couple of seconds, so it gets frequent bias refreshes for free. The V5's own IMU, by contrast, calibrates once at init and then runs open-loop as it warms up, with no further correction. Since heading error couples directly into position error, that difference matters more than raw sensor specs.

Timing tells the same story: the BNO085 free-runs at 100 Hz and the ESP32 timestamps every frame, versus a polled V5 IMU with jitter that's never observable from outside.

It also just survives the sport better: it holds up under hard impacts in a way the V5 IMU doesn't.

RVC mode specifically was chosen because it sends unsolicited 19-byte frames at 115200 baud with no host protocol overhead, and critically, it doesn't share the I²C bus with the ToF sensors, which keeps the IMU's timing clean and independent of whatever the ToF chain is doing.

Two wiring details are deliberate, not incidental: PS0 is tied to the rail via the breakout's own solder jumper rather than a GPIO, to avoid a power-sequencing race; RST goes to GPIO 38 so firmware can re-latch the mode if needed.

Two parts rejected on paper, not in practice: the ICM-42688-P and ADIS16505 both have better raw bias stability than the BNO085, but neither has onboard sensor fusion. Given how often this robot stops, which is exactly where the BNO085's SH-2 fusion shines, the plan is to keep the BNO085 and add zero-velocity updates (ZUPTs) on top, rather than chase raw IMU specs that don't matter as much for this usage pattern.

## ToF: 4× TMF8821

These exist to bound odometry drift by giving the MCL filter absolute corrections. Odometry alone only ever gets worse over time, and something has to anchor it back to ground truth.

Multizone sensing specifically matters because a single TMF8821 yields both perpendicular distance *and* wall angle from one reading. That angle constrains heading directly, which is exactly what's needed to hit the ±2° target, but it only works if the zones are pre-processed into a (distance, angle) pair before they reach the filter; feeding raw zones in as independent measurements breaks the math.

One sensor faces each direction (front, right, rear, left), so some wall is visible from almost anywhere on the field, and they're staggered at 90° phase offsets to smooth corrections over time and spread CPU load rather than hitting all four at once.

These sensors are at their best near walls at close-to-normal incidence, which happens to be exactly where Gate 2 is measured. They have no moving parts to take hits, and each reading is a single snapshot with no scan skew to correct for.

The known weaknesses are real and worth stating plainly: each zone covers roughly 10°, so readings start blending in clutter at range; readings are biased at oblique angles; and the sensor only delivers about 15 Hz in 4×4 mode. A 2D lidar (LD06 / STL-27L) is the fallback, but only gets pulled in if the characterization matrix actually fails. It's not a parallel-track option.

## The RS-485 Link

The smart port is physically RS-485 under the hood, so the ESP32 side uses an SP3485 half-duplex transceiver with hardware DE/RE control to match it. The brain stays bus master throughout, using request/response framing specifically to avoid bus contention.

The link runs at 230400 baud rather than the 460800 the latency-parity analysis assumed, because the V5's oscillator carries roughly 1.5% error at 460800 against a 2–3% total error budget, which is not enough margin to run that fast safely. `kLinkBaud` lives in `link.hpp` as the single source of truth for this, so it's worth flagging that the latency numbers quoted above were measured at 460800 and should be re-run at 230400 to confirm they still hold.
