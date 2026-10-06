# Firmware snapshots

The pod's ESP32-S3 firmware at three points in the project. Each folder is an exact copy of one commit. Nothing has been edited, so the commit hash tells you what you are looking at.

| Folder | Hardware | Commit | Written by |
|---|---|---|---|
| [`breadboard/`](breadboard/) | The breadboard build | `f9613c0` (Sep 12) | Ronan Hawkins |
| [`pcb-v1/`](pcb-v1/) | PCB V1 | `f138950` (Sep 30) | Ronan Hawkins |
| [`pcb-v2/`](pcb-v2/) | PCB V2 | `d8f95ed` (Oct 6) | Ronan Hawkins and Albin Joby |

The commits are from [`ODOM-CODE`](https://github.com/albinjoby82-ops/ODOM-CODE), which has the same history as Ronan's [`gaelforce_esp32`](https://github.com/ronanhawkins/gaelforce_esp32). The V2 snapshot is the head of [pull request 1](https://github.com/ronanhawkins/gaelforce_esp32/pull/1). It has not been tested on a built V2 board.

What changed between them is mostly the pin numbers in `include/pod_config.hpp`. The V1 and V2 firmware are not interchangeable, so flash each one only on its own board.

Each folder builds with PlatformIO (ESP-IDF) for the ESP32-S3-DevKitC-1 with the N8R8 module. The shared `gflib` library (odometry, particle filter and link protocol) is not included here, because it is not in either source repo. It is expected next to the project as `../gflib`.

The ToF firmware image in `src/tof_firmware.cpp` comes from SparkFun's Qwiic TMF882X library. The source repos do not have a licence file.
