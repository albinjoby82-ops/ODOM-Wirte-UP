# What We Built on the Breadboard

When the parts arrived, every component from section 1 was wired onto a breadboard and driven by one firmware image. This section describes what that firmware does, and nothing more. It is the code, not a set of test results.

**Source.** Ronan Hawkins' `gaelforce_esp32` at commit `f138950` (the V1 pin map), plus commit `5a9fc94`, which fixes the ToF SPAD mask order and adds a standalone TMF8821 bench test. Both are in [`ODOM-CODE`](https://github.com/albinjoby82-ops/ODOM-CODE) and in [`ronanhawkins/gaelforce_esp32`](https://github.com/ronanhawkins/gaelforce_esp32). This is the code from before the V2 PCB rework, so the pin numbers below are the V1 pins, not the PCB's.

**Build.** PlatformIO with ESP-IDF (not Arduino), on an ESP32-S3-DevKitC-1 with the N8R8 module. IDF is used because the PCNT glitch filter, the hardware RS-485 turnaround, and the EN-pin ToF enumeration are all IDF features that the Arduino layer hides. A shared library, `gflib`, holds the odometry, particle filter and link protocol. It is symlinked in, and `link.hpp` has to stay byte-identical with the V5 project that uses the same file.

## Wiring (V1 pins)

| Function | Pins |
|---|---|
| Vertical encoder A / B | GPIO 1 / 2 |
| Horizontal encoder A / B | GPIO 3 / 4 |
| BNO085 (UART-RVC, receive only) | GPIO 8 |
| RS-485 TX / RX / DE+RE | GPIO 17 / 18 / 21 |
| ToF I²C SDA / SCL (shared) | GPIO 16 / 15 |
| ToF enable (front, right, rear, left) | GPIO 11, 12, 14, 13 |

Encoders go through the SN74LVC245A level converter. The SP3485 has DE and RE tied together and driven as the UART's RTS line. Console output goes out the UART on GPIO 43/44 at 921600 baud. Pins 0/45/46, 19/20, 26-37 and 48 are left alone because they are strapping, USB, flash, PSRAM or LED pins on this module.

## Boot sequence

1. **Hold every ToF sensor in reset** before the shared I²C bus exists, so none of them answers at the default address.
2. **Read and increment a boot ID**, a counter kept in flash, which goes into every pose report so the brain can tell a reboot from a stale link.
3. **Sanity-check the wheel scale.** If counts-per-inch is not positive, the firmware aborts.
4. **Start the encoders, IMU and RS-485 link**, wait 200 ms for a few IMU frames, then take a baseline sample of each.
5. **Start odometry** from the robot as it stands.
6. **Bring up the ToFs** (below). A missing sensor is logged and tolerated, not fatal.
7. **Seed the particle filter** from the odometry pose, with the boot ID as the random seed so runs are reproducible.
8. **Warn on every boot** that the field size is still a placeholder.

## The control loop

Everything runs in one task, in a fixed 10 ms loop (100 Hz). FreeRTOS is set to a 1000 Hz tick so the 10 ms delay is ten ticks and any drift is visible in the loop timing, rather than hidden by a one-tick delay. Each pass:

1. Sample both encoders and the IMU, with one timestamp taken before any sensor is touched.
2. Drain anything that arrived on the RS-485 link since last time.
3. Integrate odometry.
4. **Predict**: feed the odometry deltas that were actually accepted into the particle filter.
5. Compute the filter's pose estimate and confidence once.
6. **Poll one ToF sensor**, and if it produced an accepted reading, **update** the filter with it.
7. If the filter has latched as diverged, **reseed from odometry**, at most once every 500 ms.
8. Build the status flags and **send a pose report** to the brain.
9. **Listen** for the brain's reply for the rest of the tick (8 ms if the brain has been replying, 1 ms if not), instead of sleeping.
10. Print one section of telemetry, then wait for the next 10 ms boundary.

## Encoders

Each encoder uses a hardware PCNT unit in full quadrature, so all four edges of each cycle are counted. That is 8192 counts per revolution for the AMT102-V switched to 2048 PPR. A 1 µs glitch filter sits in front of it.

PCNT is a 16-bit signed counter. The driver sets the limits to ±32000 with watch points exactly on them and turns on the IDF's accumulate-on-overflow mode, so the count carries past the limit instead of wrapping. The code's own comment notes that the accumulate mode does nothing without those watch points.

The pods are mounted as a diamond, at 45°. Each side has its own reversed flag.

## IMU

The BNO085 runs in UART-RVC mode at 115200 baud and the pod only listens. The parser looks for the `0xAA 0xAA` header, reads 19-byte frames, checks the checksum, and counts bad ones. Yaw is read from each frame and unwrapped across the ±180° boundary, then negated because RVC is counter-clockwise positive and `gflib` is clockwise positive. The IMU counts as healthy only if a frame has arrived in the last 50 ms.

The breakout's P0 jumper has to be bridged, or the part comes up in I²C mode and the RX pin stays silent, which looks exactly like a broken wire.

## ToF sensors

**Bring-up.** All four TMF8821s boot at address `0x41`, so they are enabled one at a time through their EN pins and moved to `0x42`–`0x45`. Each needs its firmware image loaded on every cold start. The image is the one from SparkFun's Qwiic TMF882X library. Enumeration runs at 100 kHz, then the bus is switched to 400 kHz for running. The code notes that this needs bus pull-ups of 1.5 kΩ or lower, and that the only pull-ups are the breakouts' own.

**Configuration.** Each sensor is set to a 70 ms ranging period with 1000k iterations, and starts staggered by 17 ms so at most one result is ready at a time. The custom SPAD mask is a single row of four zones, two SPADs tall and centred, selected as map 14. The comment in the code says the stock 4×4 row sat off-axis and floor-saturated at about 64 in. The zone angles off boresight are −16.22°, −5.54°, +5.54° and +16.22°. They are derived from the SPAD pitch, not measured.

**Setup order matters.** The map must be selected before the mask can be downloaded. The first attempt at mask-first failed on all four sensors on the bench. The sequence that passed is: configure, download the mask, configure again, start measuring. A sensor that enumerates but won't start is held in reset and marked absent.

**Per-reading processing.** The loop checks which sensor has a result ready and reads only that one. For each result:

1. Drop zones with zero confidence or zero distance.
2. Project each remaining zone onto the boresight by the cosine of its off-axis angle.
3. Need at least two valid zones, or the result is rejected as no-return.
4. Take the median of the projected distances.
5. **Dispersion gate:** reject if any zone is further from the median than 15% of the range, with a 2.5 in floor.
6. **Range gates:** reject beyond 137 in as out of range, and within 6 in of that as saturated. Both limits are derived from the mount height, not measured.
7. **Motion gate:** reject if the brain reports disabled or e-stop, if yaw rate exceeds 180°/s, or if drive voltage exceeds 10 V.
8. **Lag compensation:** adjust the distance for the robot's closing speed along the sensor's bearing over 35 ms, since the reading describes where the robot was.
9. **Incidence gate:** only if the filter's confidence is at least 0.3, reject if the angle between the sensor's ray and the wall's normal exceeds 60°.

An accepted reading goes to the filter as a single distance from that sensor's mount. Every rejection reason is counted per sensor.

## Particle filter

300 particles, held in internal SRAM. PSRAM is deliberately off. The filter is about 19 KB. Prediction takes the odometry deltas. Updates take one ToF reading at a time, with a 1.5 in sensor sigma and a gate that rejects readings that contradict the estimate (8 in short, 12 in long) once confidence is above 0.3.

A reseed from a received pose, or from odometry after divergence, uses a 4 in position spread and a 5° heading spread. The code comment gives the reason for the tight spread as avoiding field-symmetry ambiguity.

## The link to the brain

The pod sends a 56-byte pose report every tick. It carries the MCL pose, confidence, flags, the raw odometry pose, the boot ID, velocities, and a timestamp taken before the sensors were read. The brain replies with a 23-byte status frame carrying motor voltages and disabled and e-stop flags. A pose-set message from the brain sets odometry and reseeds the filter, and the pod holds a "pose reset" flag in its reports until one is sent, because the brain will not drive until it sees that echo.

The status flags report IMU health, each pod's plausibility, link degradation, pose reset, and whether the filter has converged. The link counts as degraded if no brain reply has arrived within 30 ms, which is three missed frames. A stale link does not stop the pod correcting its own pose.

RS-485 turnaround is done by the UART hardware in half-duplex mode, timed off the last stop bit. The code notes that software control is the classic bug, where releasing the line early truncates the last byte and late stamps on the reply. The link baud comes from `gflib` (`kLinkBaud`) and is the same constant used to compute air time.

## What it measures about itself

The firmware prints a telemetry cycle about once a second, one section per tick so the console never overruns the loop. The code comment explains why: a full dump takes long enough to overrun the delay, and the catch-up ticks would hand odometry a time step of 0 or 1 ms and turn encoder quantisation into a bogus velocity. The sections are:

- **Pose and loop:** odometry pose, raw encoder counts, IMU heading and frame counts, loop period (min, mean, max) and worst-case work time.
- **Round trip:** min, mean and max round-trip time to the brain. The exact wire time of both frames is subtracted before the remainder is halved, which the code says would otherwise bias the result by most of a millisecond.
- **Link:** CRC errors, resyncs, dropped and out-of-order frames, version and length errors, and any pose frames the UART couldn't take whole.
- **Filter:** pose, confidence, spread, effective sample size, gated readings, divergences and reseeds.
- **Cost:** mean and max time for MCL and for ToF I²C.
- **Per sensor:** address, calibration status, time to first result, the four raw zone distances and confidences, median, lag, last reject reason, and counts for every reject type and bus error.

## The bench test

`tools/tof_test` is a separate firmware that compiles the pod's own TMF8821 driver and walks the same bring-up one stage at a time, reporting which stage failed for which sensor. It first checks that the I²C lines idle high, then scans the bus, then enumerates each sensor and starts it measuring. It then streams live distances for all four, marking a sensor DEAD or STALLED if it stops, so each can be checked with a hand or a wall in front of it. It has a switch to fall back to the stock 3×3 map if the custom mask is the suspect.

## Values still marked as placeholders in the code

These are the numbers the firmware itself says are not real measurements yet:

- **Tracking wheel diameter:** 2.0 in. The code says nothing downstream is trustworthy until this is the real wheel.
- **Pod offsets from the tracking centre:** 3.0 in each.
- **ToF mount offsets:** 8.0 in each.
- **Field interior:** 72 in half-width and half-height. The firmware logs a warning about this on every boot.

## Where this differs from section 1

Section 1 gives the design intent. The breadboard firmware differs from it in four places:

- **Cores:** one task at 100 Hz, not the two-core split with 200 Hz prediction.
- **PSRAM:** present on the module but left off.
- **ToF output:** each sensor sends one median distance, not a (distance, angle) pair.
- **Encoder wrap:** handled by the IDF's accumulate mode with watch points, not a custom high-priority overflow interrupt.
