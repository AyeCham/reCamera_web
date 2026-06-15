# reCamera Web — Conveyor Object Counting

An edge-AI application for the **Seeed Studio reCamera** that detects and counts objects crossing a line on a conveyor belt. It runs as a **Node-RED** flow on the device, exposes a live **Node-RED Dashboard 2.0** UI, and ships a standalone **Conveyor History** web page that visualizes logged counting sessions and exports daily CSV reports.

## Overview

The system uses an on-device object-detection model to watch a conveyor belt. As items cross a virtual line, each "pass" is counted and grouped into sessions (start → stop). Counts are logged per item type, displayed in real time, and archived for later review.

The project has two parts:

1. **`flow.json`** — the Node-RED flow that runs on the reCamera (camera capture, model inference, line-cross counting, dashboard UI, logging, MQTT, HTTP API, and system monitoring).
2. **`index.html`** — a standalone "Conveyor History" web page that reads daily logs from cloud storage (Supabase) and renders sessions, counts, durations, and a CSV export.

## Hardware

- **Seeed Studio reCamera** (SG2002 / CV181x RISC-V SoC with onboard TPU and camera), which ships with Node-RED and the **SSCMA** (SenseCraft Model Assistant) runtime.

## Architecture

### Node-RED flow (`flow.json`)

Roughly 199 nodes organized across three tabs (**Dashboard**, **Object Counting (Controlled)**, **Flow 1**), including:

- **reCamera nodes** — `camera`, `model`, and `sscma` for capturing frames and running model inference on the TPU; a `light` node to drive the onboard LED.
- **Counting logic** — `function` nodes that track line crossings and emit `session_start`, `counted_pass`, and `session_stop` events.
- **Dashboard 2.0 UI** — gauges, charts, text, templates, pages, sliders, and a dropdown for live monitoring and control.
- **Messaging** — MQTT (`camera/mycam001/snapshot`) and WebSocket nodes for snapshots and streaming.
- **HTTP API** —
  - `POST /api/snapshot` — capture a snapshot
  - `GET /snapshot` — retrieve the latest snapshot
  - `GET /api/history` — retrieve session history
- **Logging** — `file` nodes that append events to daily log files.
- **System monitoring** — CPU load, memory, disk usage, uptime, and OS information panels.

### History page (`index.html`)

A self-contained HTML/CSS/JS page (the "Conveyor History" dashboard) that:

- Fetches a daily log file from Supabase storage:
  `…/storage/v1/object/public/history/linecross_log_<YYYY-MM-DD>.jsonl`
- Parses the JSONL event stream into sessions.
- Renders each session with start/stop times, duration, a per-item count table, and a total.
- **Auto-refreshes every 60 seconds** and shows newest sessions first.
- **Exports a daily CSV report** via the "Download Daily Report" button.

#### Log format

Each log file is **JSONL** (one JSON object per line). Event types:

| `type`          | Fields                | Meaning                          |
|-----------------|-----------------------|----------------------------------|
| `session_start` | `time`                | A counting session began         |
| `counted_pass`  | `time`, `item`        | One item crossed the line        |
| `session_stop`  | `time`                | The session ended                |

Sessions are reconstructed by walking the events in order and accumulating `counted_pass` events per `item` between a start and stop.

## Machine Learning Pipeline

The detection model is produced and deployed with the following workflow:

1. **Data annotation — [Roboflow](https://roboflow.com/)**
   Collect images of the conveyor items and annotate them (bounding boxes per item class).

2. **Model training — [Google Colab](https://colab.research.google.com/)**
   Train the object-detection model in Colab using the exported Roboflow dataset.

3. **Model conversion — TPU-MLIR**
   Convert the trained model into the reCamera's TPU format (`cvimodel`) using SOPHGO's **TPU-MLIR** toolchain.

   Install via pip:
   ```bash
   pip install tpu_mlir[all]==1.7
   ```

   Or clone and build from source:
   ```bash
   # Clone and build TPU-MLIR
   git clone -b v1.7 --depth 1 https://github.com/sophgo/tpu-mlir.git
   cd tpu-mlir
   source ./envsetup.sh
   ./build.sh
   ```

4. **Deployment**
   Copy the converted model onto the reCamera and point the `model` node in `flow.json` at it.

> Running TPU-MLIR inside the official SOPHGO Docker image is recommended to keep the toolchain dependencies isolated.

## Getting Started

### 1. Import the flow

1. Open the reCamera's Node-RED editor in your browser.
2. Use **Menu → Import** and paste the contents of `flow.json` (or import the file).
3. Deploy the flow.

### 2. Configure

- Ensure the `model` node references your converted `cvimodel`.
- Set your MQTT broker / Supabase credentials where applicable.
- Adjust the line position and counting parameters in the function nodes to match your camera view.

### 3. View history

Host `index.html` (any static host, or open locally) and update the `SUPABASE_BASE` constant at the top of the script to point at your own Supabase storage bucket. The page then loads `linecross_log_<date>.jsonl` for the selected date.

## Project Structure

```
reCamera_web/
├── flow.json     # Node-RED flow (device app + dashboard)
├── index.html    # Standalone Conveyor History web page
└── README.md
```

## Notes

- The Supabase URL in `index.html` is a deployment-specific endpoint — replace it with your own bucket.
- Item labels shown in the history table come directly from the model's class names (the `item` field of each `counted_pass` event).

## Author

Aye Chan Aung
