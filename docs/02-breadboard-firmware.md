# What We Built on the Breadboard

When the parts arrived, we wired every component from section 1 onto a breadboard and flashed one firmware image to the ESP32-S3.

## What it does

The firmware reads the two AMT102-V encoders (through the level converter), the BNO085 IMU, and the four TMF8821 ToF sensors. It runs a 300-particle Monte Carlo localization filter and sends the resulting pose to the V5 brain over RS-485 every 10 ms (100 Hz). The brain replies with its status, and the pod uses that reply to measure round-trip time and to stop trusting ToF readings while the robot is driving hard. The pod drives nothing. It only reports pose.

Everything runs in a single task, and the firmware prints its own timing, link and per-sensor diagnostics about once a second. A separate bench test (`tools/tof_test`) brings up the four ToF sensors one stage at a time and streams live distances, so each can be checked on its own.

## The code we flashed

| What | Where |
|---|---|
| Firmware | [`ODOM-CODE` at `5a9fc94`](https://github.com/albinjoby82-ops/ODOM-CODE/tree/5a9fc94) |
| Same code, original repo | [`gaelforce_esp32`](https://github.com/ronanhawkins/gaelforce_esp32), Ronan's `main` (`f138950`) plus the ToF fix and bench test (`5a9fc94`, from PR #1) |
| Pin numbers | `include/pod_config.hpp` (V1 pins, not the PCB's) |
| Main loop | `src/main.cpp` |
| ToF bring-up and filtering | `src/tof_array.cpp` |
| ToF bench test | `tools/tof_test/` |

Build with PlatformIO (ESP-IDF) for the ESP32-S3-DevKitC-1 with the N8R8 module. The shared `gflib` library (odometry, particle filter, link protocol) is not in either repo.

## How it differs from section 1

Section 1 is the design intent. The breadboard build runs as one task at 100 Hz rather than the two-core split, leaves PSRAM off, and gives the filter one median distance per ToF sensor rather than a distance-and-angle pair.
