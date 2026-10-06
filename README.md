# ODOM POD Write-Up

The story of the ODOM POD: an ESP32-S3 odometry and localization pod that feeds pose to a VEX V5 brain over an RS-485 link.

This repo holds the narrative write-up. The operations team will turn it into a notebook, so each section is a self-contained, numbered document in `docs/`, meant to be read in order as one story.

## Reading order

| # | Section | Status |
|---|---------|--------|
| 1 | [Why These Components](docs/01-component-rationale.md) | Draft |
| 2 | [What We Built on the Breadboard](docs/02-breadboard-firmware.md) | Draft |
| 3 | [PCB V1](docs/03-pcb-v1.md) | Draft |

More sections will be added to this table as the story grows.

## What's in the repo

| Folder | What |
|---|---|
| [`docs/`](docs/) | The write-up, one numbered file per section, with images in `docs/img/` and clips in `docs/media/` |
| [`hardware/`](hardware/) | KiCad design files, Gerbers and PDFs for each PCB version |
| [`code/`](code/) | The pod firmware at each stage: breadboard, PCB V1 and PCB V2 |

## The story in one paragraph

The V5 brain is a good motor controller but a poor sensor platform. The ODOM POD moves sensing and pose estimation onto an ESP32-S3 and sends pose to the V5 as a pure sensor source. Motion control stays on the V5.

## Conventions

- One section per file in `docs/`, named `NN-short-title.md`, so the order is obvious and maps cleanly to notebook sections.
- Legality (VUR12) and latency points are woven into the relevant section, not split out.
- Figures and images go in `docs/img/` when they are added.
