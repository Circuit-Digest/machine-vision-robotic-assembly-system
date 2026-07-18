<div align="center">

# 🤖 Machine Vision-Based Robotic Assembly System Using MechArm 270 and Jetson Nano

**A real-time computer-vision system that drives a 6-DOF robotic arm + linear rail to autonomously assemble two toy cars, verifying every part placement using black-pixel ROI detection on an on-board webcam.**

[![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?logo=python&logoColor=white)](https://python.org)
[![Flask](https://img.shields.io/badge/Flask-2.3%2B-000000?logo=flask&logoColor=white)](https://flask.palletsprojects.com)
[![OpenCV](https://img.shields.io/badge/OpenCV-4.8%2B-5C3EE8?logo=opencv&logoColor=white)](https://opencv.org)
[![SocketIO](https://img.shields.io/badge/Socket.IO-5.x-010101?logo=socket.io&logoColor=white)](https://flask-socketio.readthedocs.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

<br><br>
<img src="Images/vision_demo.gif" width="720" alt="Robotic Assembly Line Vision Demo">

</div>

---

## 📖 Table of Contents

1. [Project Overview](#-project-overview)
2. [Key Features](#-key-features)
3. [System Architecture](#-system-architecture)
4. [Hardware Requirements](#-hardware-requirements)
5. [Software Requirements & Installation](#-software-requirements--installation)
6. [Configuration](#-configuration)
7. [Running the Server](#-running-the-server)
8. [Dashboard Guide](#-dashboard-guide)
9. [Calibration Workflow](#-calibration-workflow)
10. [Vision Pipeline](#-vision-pipeline)
11. [Inspection Stations](#-inspection-stations)
12. [Waypoint System](#-waypoint-system)
13. [Socket.IO Event Reference](#-socketio-event-reference)
14. [Project Structure](#-project-structure)
15. [Troubleshooting](#-troubleshooting)
16. [Contributing](#-contributing)
17. [License](#-license)

---

## 🎯 Project Overview

This system orchestrates a **MechArm 270** 6-DOF robotic arm mounted on a **stepper-driven linear rail** to perform automated, vision-verified pick-and-place assembly of two toy cars.

The arm visits five physical stations. At each inspection station it positions its **wrist-mounted webcam** over a part tray, and a real-time OpenCV pipeline **verifies the correct component is present** using black-pixel ROI analysis before the arm is allowed to pick it up. A missing part immediately halts the sequence, triggers a **physical buzzer alert**, and waits for an operator to place the correct component.

The entire stack — robot control, vision processing, rail control, and web dashboards — runs as a **single Python process on port 5000**, eliminating all inter-service network latency.

---

## ✨ Key Features

| Feature | Description |
|---|---|
| 🎥 **Live MJPEG Stream** | Real-time webcam feed with annotated ROI overlays served to the browser |
| 🔍 **Black-Pixel ROI Detection** | Station-aware part detection using calibrated bounding boxes and HSV black masking |
| 🤖 **Wi-Fi Arm Control** | Non-blocking TCP socket control of MechArm 270 via `pymycobot` |
| 🚂 **ESP32 Rail Control** | HTTP-driven stepper linear rail with 5 presets (P1–P5) |
| 🚦 **Physical Indicators** | Green/Red LED + buzzer relay control for workshop-audible alerts |
| ✅ **Assembly Checklist** | Live progress bar and per-part checklist updating on every verification |
| 🖥️ **Three Dashboards** | Home display, Operator dashboard, Waypoint editor — all browser-based |
| 🛡️ **Sim Mode Fallback** | Full simulation when no hardware is connected — no code changes needed |
| 💾 **JSON Persistence** | All waypoints, poses, ROIs, and network config survive server restarts |

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│            main.py  (single process, port 5000)       │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────────┐ │
│  │ RobotManager │   │  RailManager │   │  RailIndicatorMgr   │ │
│  │  Wi-Fi TCP   │   │  HTTP / ESP32│   │  HTTP relay control  │ │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬──────────┘ │
│         │                  │                        │            │
│  ┌──────▼──────────────────▼───────────────────────▼──────────┐ │
│  │                   PlaybackEngine                            │ │
│  │  Sequences arm + rail movements + vision verification       │ │
│  └──────────────────────────┬──────────────────────────────────┘ │
│                             │  shared in-process memory           │
│  ┌──────────────────────────▼──────────────────────────────────┐ │
│  │             vision_thread()  (~30 FPS)                      │ │
│  │   OpenCV: Black-pixel ROI detection per station             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Flask Routes + Socket.IO  →  Browser Dashboards                │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Camera → vision_thread → stable_detected_parts (dict)
                              ↓ (zero-latency in-process read)
                         PlaybackEngine → verifies part → continues sequence
                              ↓
                         Socket.IO → Browser (checklist, log, progress)
```

---

## 🔧 Hardware Requirements

| Component | Specification | Notes |
|---|---|---|
| **Robot Arm** | Elephant Robotics MechArm 270 | Wi-Fi model required |
| **Linear Rail** | Custom stepper rail with ESP32-C6 | 5 presets: P1–P5 |
| **Camera** | USB webcam (640×480 minimum) | Wrist-mounted on arm |
| **Relay Board** | 3-channel relay (12V) | Buzzer + Green/Red LED |
| **Network** | Wi-Fi AP or router | Arm, ESP32, and host PC on same LAN |
| **Host Computer** | Any x86/ARM Linux or Windows PC | Python 3.9+ required |

> [!NOTE]
> The system runs in full **simulation mode** if no hardware is connected. The camera stream is replaced by an animated mock feed, all Socket.IO events fire normally, and you can test the complete UI and checklist flow without any physical hardware.

---

## 💻 Software Requirements & Installation

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/machine-vision-robotic-assembly-system.git
cd machine-vision-robotic-assembly-system
```

### 2. Create a virtual environment (recommended)

```bash
python -m venv venv

# Windows
venv\Scripts\activate

# Linux / macOS
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### Dependency Overview

| Package | Purpose |
|---|---|
| `flask` | Web server & template rendering |
| `flask-socketio` | Real-time WebSocket transport |
| `requests` | HTTP calls to the ESP32 Rail API |
| `pymycobot` | MechArm 270 Wi-Fi TCP SDK |
| `opencv-python` | Video capture, black-pixel ROI masking, MJPEG stream |
| `numpy` | Array operations for vision pipeline |

---

## ⚙️ Configuration

### Step 1 — Set your network IPs

Edit **`config.py`** and update the two IP constants at the bottom of the file:

```python
ARM_IP         = "192.168.1.100"   # Your MechArm270's Wi-Fi IP address
RAIL_IP        = "192.168.1.101"   # Your ESP32-C6 Rail controller IP
```

> [!TIP]
> You can also set these at runtime **without restarting the server** — open the Operator Dashboard, click the gear icon ⚙️, and enter your IPs. They are saved to `robot_config.json` and `rail_config.json` and take effect immediately.

### Step 2 — Camera index

In `config.py`, set the correct USB camera index:

```python
CAMERA_INDEX = 0    # 0 = default (first USB camera)
                    # 1 = second USB camera, etc.
```

### Step 3 — Vision detection thresholds

Tune these in `config.py` to match your lighting and part colours:

```python
# Black part detection
BLACK_HSV_LOWER = (0,   0,   0)    # HSV lower bound for black
BLACK_HSV_UPPER = (180, 255, 50)   # V < 50 = genuinely dark/black
BLACK_PIXEL_RATIO = 0.10           # 10% of ROI must be black to count as "present"

# Temporal hysteresis (noise suppression)
CONFIRM_FRAMES   = 5    # frames a part must be seen before "stable"
DISAPPEAR_FRAMES = 10   # frames a part must be missing before "lost"
```

### Step 4 — Runtime-persisted configuration files

The following JSON files are auto-generated and updated by the dashboards. They override the `config.py` defaults:

| File | Contents | Editable via |
|---|---|---|
| `robot_config.json` | Arm IP, port, axis calibration offsets | Operator Dashboard ⚙️ |
| `rail_config.json` | Rail ESP32 IP | Operator Dashboard ⚙️ |
| `camera_settings.json` | Brightness, contrast, exposure, gain | Operator Dashboard 🎥 |
| `scan_poses.json` | 6-joint arm angles per inspection station | Waypoint Editor → Scan Poses |
| `roi_boundaries.json` | Per-part pixel bounding boxes (ROI) | Operator Dashboard → ROI Editor |
| `zone_boundaries.json` | Spatial zone definitions per station | Operator Dashboard → Zone Editor |
| `waypoints.json` | Full arm + rail motion sequence | Waypoint Editor |

> [!WARNING]
> The `waypoints.json` included in this repository was recorded on a **specific physical rig**. Joint angles and positions will **not transfer** to a different robot or table layout. You must re-record all waypoints using the Waypoint Editor for your own setup.

---

## 🚀 Running the Server

```bash
python main.py
```

The server starts on `http://0.0.0.0:5000`. Open a browser and navigate to:

| URL | Dashboard |
|---|---|
| `http://localhost:5000/` | 🏠 Home Display Dashboard |
| `http://localhost:5000/camera_view` | 🖥️ Operator Dashboard |
| `http://localhost:5000/editor` | ✏️ Waypoint Editor |

On startup you will see:

```
======================================================================
  Robotic Assembly System (MechArm 270 & Jetson Nano) -> http://localhost:5000
  Robotic Arm + Stepper Linear Rail + Black-pixel ROI Vision Integrated.
======================================================================
```

The robot arm and rail connections are made in the background — if they fail the server continues in simulation mode and retries every 15 seconds.

---

## 🖥️ Dashboard Guide

### 🏠 Home Display Dashboard (`/`)

Clean status overview, suitable for display on a monitor during operation.

```
┌──────────────────────────┬──────────────────────┐
│   Live Camera Stream     │   System Info         │
│   (annotated MJPEG)      │   Robot: MechArm 270  │
│                          │   Detection: Black ROI│
├──────────────────────────┼──────────────────────┤
│   Last Detection         │  Assembly Progress   │
│   Snapshot               │  ████████░░░  75%    │
│                          │                      │
│                          │  ✅ Chassis 1         │
│                          │  ✅ Chassis 2         │
│                          │  ✅ WheelBase 1a      │
│                          │  ⏳ WheelBase 2a      │
│                          │  ...                  │
│                          │                      │
│                          │  Sensor: 24.3°C      │
│                          │  Door: CLOSED        │
│                          │                      │
│                          │  System Log          │
│                          │  ✓ Chassis 1 confirmed│
└──────────────────────────┴──────────────────────┘
```

- **▶ Play Sequence** — starts the full automated assembly run
- **⏹ Stop Arm** — emergency halt

### 🖥️ Operator Dashboard (`/camera_view`)

Full-featured operator view for monitoring and controlling the assembly:

- **Live annotated stream** — ROI boxes (grey → yellow → green), part labels, pixel-ratio readout, station overlay
- **Robot telemetry** — 6 joint angles, XYZ coordinates updated every 200 ms
- **Assembly checklist** — all parts for both cars, updating in real time
- **8-Switch Simulator Panel** — toggle any part as "present" for dry-run testing without physical components
- **Halt/Error Panel** — if the sequence stops on a missing part, this panel shows the reason and a restart button

### ✏️ Waypoint Editor (`/editor`)

Engineering tool for teaching the robot its motion sequence:

1. **Jog controls** — move each of the 6 joints or X/Y/Z coordinates from the browser
2. **Record** — save the current arm pose as a new waypoint (angles, coords, or both)
3. **Edit sequence** — rename, reorder, adjust speed, vacuum action, delay, and rail preset per waypoint
4. **Scan Pose console** — jog the arm to the perfect camera inspection angle, then click **Set P1** (or P2/P4/P5) to save to `scan_poses.json`
5. **Motor calibration** — lock/unlock individual joints, adjust motor power levels
6. **Network settings** — change Arm IP, Rail IP, and speed without restarting the server

---

## 📐 Calibration Workflow

Follow this sequence when setting up on a new physical rig:

```
1. Physical setup
   └─ Mount arm on rail, position part trays at stations P1, P2, P4, P5

2. Network configuration
   └─ Set ARM_IP and RAIL_IP in config.py or via Operator Dashboard ⚙️

3. Camera verification
   └─ Open /camera_view, check live stream
   └─ Adjust CAMERA_INDEX in config.py if wrong camera opens

4. Scan pose recording  (Waypoint Editor → 📸 Inspection Scan Poses)
   └─ Jog arm to view P1 tray → click "Set P1 (Chassis)"
   └─ Repeat for P2, P4, P5

5. ROI calibration  (Operator Dashboard → ROI Editor)
   └─ Move arm to scan pose for each station
   └─ Draw bounding boxes around each part slot on the video frame
   └─ Save → updates roi_boundaries.json

6. Detection threshold tuning  (config.py)
   └─ Adjust BLACK_PIXEL_RATIO until parts detect cleanly
   └─ Use 8-Switch Simulator to verify checklist behaviour

7. Waypoint recording  (Waypoint Editor)
   └─ Record pick positions at each station
   └─ Record place positions at P3 (assembly zone)
   └─ Add vision_check waypoints before each pick
   └─ Test with "Skip Vision" mode first, then enable vision checks

8. Full test run
   └─ Place parts at all stations
   └─ Click ▶ Play Sequence on Home Dashboard
   └─ Verify checklist completes 100%
```

---

## 👁️ Vision Pipeline

The vision system runs in a **dedicated background thread** at ~30 FPS using OpenCV.

### Black-Pixel ROI Detection

The only detection method used is black-pixel ratio analysis within calibrated Region of Interest (ROI) bounding boxes:

```
Frame (BGR)
  │
  ├─ GaussianBlur (5×5) → HSV conversion
  │
  └─ For each active part at current station:
        Crop frame to ROI bounding box  (from roi_boundaries.json)
        Apply black HSV mask: H[0-180] S[0-255] V[0-50]
        ratio = countNonZero(mask) / roi_area
        present = ratio - ambient_baseline ≥ BLACK_PIXEL_RATIO (10%)
```

An **adaptive ambient baseline** is captured on the first frame after the arm settles at its scan pose. This compensates automatically for dim environments or dark surfaces in the background.

### How Detection Works Per Station

1. Rail moves arm to station (e.g. P1)
2. Arm commands itself to pre-saved scan pose for that station (`scan_poses.json`)
3. Vision thread activates and begins checking only the parts registered to that station (`STATION_PARTS` in `config.py`)
4. For each active part slot, it crops the frame to the ROI (`roi_boundaries.json`) and measures the black-pixel ratio
5. Parts meeting the threshold are accumulated over frames; confirmed after `CONFIRM_FRAMES` consecutive detections

### Hysteresis Filtering

Raw frame results are not immediately acted upon — a two-stage filter prevents false positives from motion blur, shadows, or lighting glitches:

```
MISSING ──[seen for 5 consecutive frames]──→ STABLE (detected)
STABLE  ──[missing for 10 consecutive frames]──→ MISSING
```

### Stream Annotation

During an active vision check, the live stream shows:

- **Grey boxes** — ROI is active, part not detected
- **Yellow boxes** — part seen but not yet stable (accumulating frames)
- **Green boxes** — part confirmed stable (checkmark ✓ + ratio %)
- **Station panel** — current station label + list of active part slots

---

## 🏭 Inspection Stations

| Station | Preset | Detection Method | Parts Verified |
|---|---|---|---|
| **P1** | `P1 (Chassis)` | Black-pixel ROI | Chassis 1, Chassis 2 |
| **P2** | `P2 (Wheelbase)` | Black-pixel ROI | WheelBase 1a, 2a, 1b, 2b |
| **P3** | `P3 (Assemble)` | *(none — assembly zone)* | — |
| **P4** | `P4 (Wheels)` | Black-pixel ROI (8 slots) | Wheel Slots 1–8 |
| **P5** | `P5 (Body)` | Black-pixel ROI | Body 1, Body 2 |

---

## 📋 Waypoint System

### Waypoint Record Format

```json
{
  "name": "Pick Chassis 1",
  "type": "angles",
  "data": [15.2, -45.1, 90.0, 0.0, 45.0, 0.0],
  "speed": 40,
  "vacuum": "on",
  "vacuum_delay_ms": 300,
  "delay_ms": 200,
  "rail_preset": "P1"
}
```

| Field | Values | Description |
|---|---|---|
| `type` | `angles`, `coords`, `both`, `vision_check` | Movement type |
| `data` | `[j1, j2, j3, j4, j5, j6]` or `[x, y, z, rx, ry, rz]` | Target position |
| `vacuum` | `"on"`, `"off"`, `"none"` | Vacuum pump action after movement |
| `rail_preset` | `"P1"`–`"P5"`, `"NONE"` | Rail position to move to after arm arrives |
| `speed` | `1`–`100` | Arm movement speed percentage |

### `vision_check` Waypoint

A special waypoint type that **does not move the arm or rail**. It triggers a vision verification cycle:

```json
{
  "name": "Verify Chassis 1",
  "type": "vision_check",
  "rail_preset": "P1",
  "check_part": "Chassis 1"
}
```

When the PlaybackEngine reaches this waypoint it:
1. Commands the arm to the pre-saved scan pose for that station
2. Waits 1.5 s for mechanical settling
3. Activates vision detection for the named part
4. Waits until the part is stably detected (or halts on timeout/stop)
5. Emits `part_verified` to update the UI checklist

---

## 📡 Socket.IO Event Reference

### Server → Client

| Event | Key Payload Fields | Description |
|---|---|---|
| `robot_status` | `angles[]`, `coords[]`, `rail{}`, `vision_detected[]` | Full telemetry broadcast (every 200 ms) |
| `status` | `state`, `step`, `total`, `message` | Playback state machine update |
| `playback_progress` | `index`, `total`, `name`, `state` | Per-waypoint progress update |
| `part_verified` | `part`, `rail`, `snapshot_ts` | Part confirmed — update checklist |
| `vision_log` | `level`, `message` | Colour-coded log line for consoles |
| `detection_started` | `part`, `station` | Vision scan is active |
| `detection_stopped` | `part`, `station`, `found` | Vision scan ended |
| `snapshot_updated` | *(none)* | Last-detection image on disk has changed |
| `error` | `part`, `position`, `message` | Emergency stop / verification fault |
| `command_complete` | `action`, `cmd_id` | Arm command acknowledged by hardware |

### Client → Server

| Event | Payload | Description |
|---|---|---|
| `play_sequence` | `start_index`, `speed`, `skip_vision` | Start playback |
| `stop_sequence` | *(none)* | Emergency stop |
| `toggle_mock` | `part`, `value` | Toggle hardware simulator switch |
| `jog_joint` | `joint`, `delta`, `speed` | Move one joint by delta degrees |
| `jog_coord` | `axis`, `delta`, `speed` | Move in Cartesian space |
| `record_waypoint` | `type`, `name`, `vacuum`, `rail_preset`, `speed` | Save current pose |
| `set_scan_pose` | `station` | Save current arm angles as scan pose |
| `update_ip` | `arm_ip`, `rail_ip` | Change network config without restart |

---

## 📁 Project Structure

```
machine-vision-robotic-assembly-system/
│
├── main.py                    # Main Flask + SocketIO application (2,700+ lines)
├── config.py                  # Shared constants: IPs, HSV thresholds, station maps
├── requirements.txt           # Python dependencies
├── .gitignore                 # Excludes __pycache__, logs, runtime snapshots
├── LICENSE                    # MIT License
│
├── templates/
│   ├── home.html              # 🏠 Home display (live feed + status checklist)
│   ├── operator_view.html     # 🖥️  Operator Dashboard (full telemetry + controls)
│   └── waypoint_editor.html   # ✏️  Waypoint Editor & Scan Pose console
│
├── static/
│   └── socket.io.min.js       # Socket.IO client (served locally, no CDN dependency)
│
├── hardware/                  # 🔌 PCB & Schematic Design Files
│   ├── schematic.pdf          # PCB schematic diagram (PDF)
│   ├── gerbers.zip            # Manufacturing-ready Gerber files
│   ├── controller.kicad_sch   # KiCad Schematic design
│   ├── controller.kicad_pcb   # KiCad PCB board layout
│   ├── controller.kicad_pro   # KiCad Project file
│   └── controller.kicad_prl   # KiCad Project local settings
│
├── Images/                    # 🖼️ Documentation Media Files
│   ├── vision_demo.gif        # Animated GIF showing the real-time vision tracking
│   ├── pcb_render.png         # 3D render of the custom PCB board
│   ├── pcb_layout.png         # 2D layout layout of the PCB routing
│   ├── control_panel.jpg      # Physical photo/render of the control panel
│   └── cad_render.png         # 3D CAD rendering of the full assembly layout
│
├── waypoints.json             # Recorded motion sequence (re-record for your rig)
├── scan_poses.json            # Camera inspection arm poses per station
├── roi_boundaries.json        # Calibrated per-part ROI bounding boxes (pixels)
├── zone_boundaries.json       # Spatial zone definitions per station
├── camera_settings.json       # USB camera brightness/contrast/exposure settings
├── robot_config.json          # Runtime arm IP config (overrides config.py)
└── rail_config.json           # Runtime rail IP config (overrides config.py)
```

---

## 🔧 Troubleshooting

### Robot arm won't connect

- Confirm the MechArm 270 is powered on and joined to your Wi-Fi network
- Check `ARM_IP` in `config.py` or `robot_config.json` matches the arm's actual IP
- The arm's IP is shown on the robot's display or in your router's DHCP table
- The server retries the connection every 15 seconds — check the console for `Wi-Fi connect failed` messages

### Rail won't move

- Confirm the ESP32-C6 board is powered and on the same LAN
- Check `RAIL_IP` in `config.py` or `rail_config.json`
- Open `http://YOUR_RAIL_IP/status` in a browser — you should get a JSON response

### Camera not opening / black screen

- Try changing `CAMERA_INDEX` in `config.py` (0, 1, 2 …)
- On Linux, if using V4L2: ensure `CAMERA_BACKEND = "CAP_V4L2"` in `config.py`
- Check that no other application has the camera locked

### Parts not being detected

- Verify the arm reaches the correct scan pose (check `scan_poses.json`)
- Use the **ROI Editor** in the Operator Dashboard to confirm the bounding boxes are positioned over the part slots
- Lower `BLACK_PIXEL_RATIO` in `config.py` if detection is too strict
- Check ambient lighting — very bright or uneven light can wash out the black pixel mask

### Detection works in sim but not on hardware

- Verify `roi_boundaries.json` is calibrated for your actual camera view angle (sim uses default zones)
- Re-run the scan pose recording for each station in the Waypoint Editor

---

## 🤝 Contributing

Contributions, bug reports, and feature suggestions are welcome!

1. **Fork** the repository
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Commit your changes: `git commit -m "feat: add your feature description"`
4. Push to your fork: `git push origin feature/your-feature-name`
5. Open a **Pull Request** against the `main` branch

### Code Style

- Python: follow [PEP 8](https://pep8.org/)
- Docstrings: include for all new public functions and classes
- Existing `# ─────` section dividers should be preserved for consistency

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

---

<div align="center">

Built with ❤️ using **Flask · OpenCV · pymycobot · Socket.IO**

</div>
