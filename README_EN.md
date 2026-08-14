# reBot-Isaacsim

[简体中文](./README.md) | English | [Español](./README_ES.md)

reBot-Isaacsim is an NVIDIA Isaac Sim project targeting the reBotDM robot. The code in this repository is currently compatible only with reBotDM and may not work with other reBotArm / RS variants.

Important notes:

- Compatibility: reBotDM only.
- Recommended Isaac Sim version: 6.0.0.
- Recommended installation method: download the official Isaac Sim zip release and extract it locally.
- All scripts that interact directly with Isaac Sim must be launched using the official `python.sh` from the Isaac Sim release.
- Real-robot sender scripts run inside the workspace `uv` environment (see `third_party/reBotArm_control_py`).

## Component Overview

This project provides several sender variants for different use cases:

| Component | Description |
|------|------|
| `gravity_joint_sender` | Gravity-compensation / handle mode: for modified robots (gripper removed, handle attached). Allows hand-guiding and streams joint angles to Isaac Sim. |
| `isaacsim_ik_sender` | Inverse-Kinematics (IK) mode: input end-effector pose, solve IK and send joint angles to Isaac Sim. |
| `isaacsim_traj_sender` | Trajectory (Traj) mode: IK + joint-space trajectory generation (MIN_JERK) for smooth motions. |
| `isaacsim_joint_test_sender` | Joint-test mode: no hardware required; sends preset joint trajectories to verify receiver and communication. |
| `joint_reader_sender` | Real-to-Sim mapping: read-only joint-angle mirroring for visualization alongside other control projects. |

## System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         reBot-Isaacsim                           │
│                                                                  │
│   ┌──────────────────────┐        ┌─────────────────────────┐    │
│   │ Sender (Terminal 2)  │  UDP   │   Receiver (Terminal 1)│    │
│   │                      │  JSON  │                         │    │
│   │ gravity_joint_sender │──────▶ │ isaacsim_joint_receiver │   │
│   │                      │ 5005   │                         │    │
│   │  • reBotArm_control  │        │  • Isaac Sim            │    │
│   │    _py uv env        │        │  • Ground + robot USD   │    │
│   │  • MIT + gravity FF  │        │  • Joint-angle sync     │    │
│   │  • Hand-guided OK    │        │  • Gripper dual-joint   │    │
│   └──────────────────────┘        └─────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

## Directory Layout

```
reBot-Isaacsim/
├── pyproject.toml                           # uv workspace configuration
├── README.md
├── README_EN.md
├── README_ES.md
├── reBotArm_Isaacsim/                       # Main example directory
│   ├── gravity_joint_sender.py              # Physical-robot sender (run inside uv)
│   ├── isaacsim_ik_sender.py                # IK sender (launch with Isaac's python.sh)
│   ├── isaacsim_traj_sender.py              # Trajectory sender (launch with Isaac's python.sh)
│   ├── isaacsim_joint_test_sender.py        # Test sender (use python.sh if it imports Isaac Sim)
│   ├── joint_reader_sender.py               # Read-only mapping sender
│   ├── isaacsim_joint_receiver.py           # Isaac Sim receiver (launch with python.sh)
│   ├── live_sync.py                         # Launch guidance script
│   └── ...
├── third_party/
│   └── reBotArm_control_py/                 # Robot control library (independent uv env)
│       ├── pyproject.toml
│       └── ...
├── urdf/
│   └── ...                                  # Robot URDF / configuration files
├── usd/
│   └── reBot_B601_DM/
│       └── reBot_B601_DM.usda               # reBotDM asset
└── ...
```

Note: run the Python modules directly and choose the correct launcher:

- Use `uv run python ...` for scripts that run in the project `uv` environment (real-robot senders).
- Use `"$ISAACSIM_ROOT/python.sh"` for scripts that require Isaac Sim's native runtime.

## Dependencies and Prerequisites

| Component | Requirement |
|------|------|
| Robot model | Current code is compatible with reBotDM only |
| Isaac Sim | 6.0.0; official zip release recommended |
| Isaac Sim path | Recommended to set `ISAACSIM_ROOT` to the extracted release folder |
| Serial/CAN interface | Default is `ttyACM0`, configurable in `third_party/reBotArm_control_py/config/rebotarm_dm.yaml` |
| Python | Sender scripts run under Python 3.10+ managed by `uv`; Isaac Sim runtime is provided by `python.sh` |
| uv | Recommended for managing project dependencies |
| reBotArm_control_py | Run `uv sync` inside `third_party/reBotArm_control_py` |

### Recommended Isaac Sim installation

Download the official Isaac Sim zip release, extract it to a stable location, then set `ISAACSIM_ROOT` to the release folder. Example:

```bash
mkdir -p /home/seeed/IsaacSim
# unzip the official Isaac Sim zip into that directory
# after extraction you should find something like:
# /home/seeed/IsaacSim/_build/linux-x86_64/release/python.sh

export ISAACSIM_ROOT=/home/seeed/IsaacSim/_build/linux-x86_64/release
```

### Check serial/CAN interface

```bash
# List USB-serial ports (ensure the USB2CAN port appears)
ls /dev/ttyACM*

# Grant permissions if necessary
sudo chmod 666 /dev/ttyACM*
```

## Environment Setup

### 1. Install Isaac Sim 6.0.0

Use the official zip release and verify `python.sh` exists in the release folder.

### 2. reBotArm_control_py environment

```bash
cd third_party/reBotArm_control_py
uv sync
```

## Launch (Two-Terminal Mode)

Two independent terminals are required. Terminal 1 is the Isaac Sim receiver, Terminal 2 is the real-robot sender. You can run both locally by setting `DEFAULT_SIM_HOST` and `DEFAULT_REBOT_ARM_HOST` to `127.0.0.1` in the scripts.

### Terminal 1 — Launch the Isaac Sim receiver

Start the receiver with Isaac Sim's `python.sh`:

```bash
"/path/to/your/isaacsim/release/python.sh" \
    /path/to/reBot-Isaacsim/reBotArm_Isaacsim/isaacsim_joint_receiver.py
```

Expected behavior:

- Isaac Sim GUI launches.
- Ground and reBotDM USD asset load.
- The receiver listens on UDP `DEFAULT_SIM_HOST:5005`.
- The receiver waits for the sender to connect.

### Terminal 2 — Launch the real-robot sender

Start the sender after the receiver is running:

```bash
cd /path/to/reBot-Isaacsim
uv run python reBotArm_Isaacsim/gravity_joint_sender.py
```

Expected behavior:

- The physical robot connects and MIT + gravity feed-forward is enabled.
- The robot can be hand-guided.
- Joint angles are streamed over UDP at ~60 Hz.

### Other Isaac Sim-related scripts

Launch other Isaac Sim scripts with `python.sh`:

```bash
export ISAACSIM_ROOT=/home/seeed/IsaacSim/_build/linux-x86_64/release
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_traj_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_test_sender.py
```

Notes:

- `gravity_joint_sender.py` and `joint_reader_sender.py` are typically run inside the project `uv` environment.
- `isaacsim_*` scripts must be launched with Isaac Sim's `python.sh` so that `SimulationApp` and the native runtime initialize correctly.

### IK / Traj input formats (examples)

```
x y z                       # position (m), keep orientation unchanged
x y z r p y                 # position + orientation (m/deg)
q j1 j2 j3 j4 j5 j6         # direct joint angles (deg)
gripper <0~1>                # update gripper only
```

Trajectory mode:

```
x y z
x y z r p y
q j1 j2 j3 j4 j5 j6
gripper <0~1>
speed <scale>
resync
```

### Read-only mapping mode (`joint_reader_sender`)

Run:

```bash
cd /path/to/reBot-Isaacsim
uv run python reBotArm_Isaacsim/joint_reader_sender.py
```

Expected behavior:

- Reads joint angles in passive feedback mode only (no control commands sent).
- Streams joint angles over UDP at ~60 Hz.
- Mirrors physical-arm motion into Isaac Sim for visualization while another project controls the robot.

## Communication Protocol

UDP JSON on `127.0.0.1:5005`.

Per-frame payload example:

```json
{
  "sequence": 123,
  "timestamp": 1718000000.123,
  "joint_positions": [0.0, 0.1, 0.2, -0.1, 0.0, -0.02],
  "gripper_position": 0.05
}
```

| Field | Type | Description |
|------|------|------|
| `sequence` | int | Monotonically increasing sequence number |
| `timestamp` | float | Unix timestamp (seconds) |
| `joint_positions` | float[6] | First 6 joint angles (rad) |
| `gripper_position` | float | Gripper finger target position (m) |

Gripper mapping (examples):

| Sender | Mapping to `gripper_position` (m) |
|------|------|
| `gravity_joint_sender` | `gripper_q × 0.03` (`GRIPPER_POSITION_SCALE = 0.03`) |
| `joint_reader_sender` | `gripper_q × 0.007` (`GRIPPER_POSITION_SCALE = 0.007`) |
| `isaacsim_traj_sender` | `ratio × 0.045` (input `gripper <0~1>`, clipped to 0.045 m) |
| `isaacsim_ik_sender` | raw `ratio ∈ [0,1]` sent as meters |

## Configuration Parameters

### Sender (`gravity_joint_sender.py`)

| Parameter | Default | Description |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | Number of joints |
| `DEFAULT_PORT` | 5005 | UDP port |
| `DEFAULT_SEND_HZ` | 60.0 | Send frequency (Hz) |
| `GRIPPER_POSITION_SCALE` | 0.03 | Scale factor from gripper angle to position |
| `position_alpha` | 0.2 | Low-pass filter coefficient |

### Receiver (`isaacsim_joint_receiver.py`)

| Parameter | Default | Description |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | Number of joints |
| `DEFAULT_PORT` | 5005 | UDP port |
| `DEFAULT_RENDER_HZ` | 120.0 | Simulation render frequency (Hz) |
| `ROBOT_PRIM_PATH` | `/World/reBotArm` | Robot prim path inside Isaac Sim |
| `ASSET_RELATIVE_PATH` | `usd/reBot_B601_DM/reBot_B601_DM.usda` | USD asset path relative to the repo root |

## Troubleshooting

### `OSError: [Errno 98] Address already in use`

Port 5005 is already in use. Identify and stop the occupying process:

```bash
sudo lsof -i :5005
kill <PID>
```

### Isaac Sim asset not found

```bash
ls usd/reBot_B601_DM/reBot_B601_DM.usda
```

### Serial / CAN issues

```bash
can_restart can0
ip -details link show can0 | grep bitrate
```

### Joint angles out of sync

- Confirm sender and receiver ports match (5005).
- Check the sender log for repeating `[send]` output.
- Check the receiver log for repeating `[recv]` output.
- Use `isaacsim_joint_test_sender.py` to rule out hardware issues.

## Components and Python Environments

| Component | Python environment | Launch method |
|------|------------|---------|
| Real-robot sender | Project `uv` environment + `reBotArm_control_py` | `uv run python reBotArm_Isaacsim/gravity_joint_sender.py` |
| Read-only mapping sender | Project `uv` environment | `uv run python reBotArm_Isaacsim/joint_reader_sender.py` |
| Isaac Sim receiver | Isaac Sim official Python (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_receiver.py` |
| Isaac Sim-related scripts | Isaac Sim official Python (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py` |
