# reBot-Isaacsim

[简体中文](./README.md) | English | [Español](./README_ES.md)

reBot-Isaacsim is an NVIDIA Isaac Sim project for the reBotDM robot. The current code is only compatible with reBotDM and is not suitable for other models.

> Important notes:
> - Current compatibility: reBotDM only
> - Recommended Isaac Sim version: 6.0.0
> - Recommended installation method: download the official zip archive and extract it locally
> - All scripts directly related to Isaac Sim must be launched via the official `python.sh`
> - The real robot sender still uses the repository's `uv` environment

## Component Overview

This project provides various sender modes to meet different use cases:

| Component | Description |
|------|------|
| `gravity_joint_sender` | **Gravity compensation handle mode**: for modified robots (gripper removed and handle installed), the robot can be manually moved under gravity compensation and joint angles are synchronized to Isaac Sim in real time. |
| `isaacsim_ik_sender` | **Inverse kinematics (IK) mode**: provide the end-effector pose, solve IK to obtain joint angles, and send them to Isaac Sim. |
| `isaacsim_traj_sender` | **Trajectory (Traj) mode**: on top of IK, adds joint-space trajectory planning (MIN_JERK timing profile) for smooth movement control. |
| `isaacsim_joint_test_sender` | **Joint test mode**: no real robot needed; sends preset joint-angle trajectories to verify the Isaac Sim receiver and communication. |
| `joint_reader_sender` | **Real-to-Sim mapping mode**: reads joint angles in read-only mode and maps them to Isaac Sim, suitable for use with other control projects (for example, when the real robot is doing other tasks, the same motion is mirrored for visualization in Isaac Sim). |

## System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         reBot-Isaacsim                           │
│                                                                  │
│   ┌──────────────────────┐        ┌─────────────────────────┐    │
│   │ Sender (Terminal 2)  │  UDP   │   Receiver (Terminal 1) │    │
│   │                      │  JSON  │                         │    │
│   │ gravity_joint_sender │──────▶ │ isaacsim_joint_receiver │    │
│   │                      │ 5005   │                         │    │
│   │  • reBotArm_control  │        │  • Isaac Sim            │    │
│   │    _py uv env        │        │  • Ground + robot USD   │    │
│   │  • MIT + gravity FF  │        │  • Joint-angle sync     │    │
│   │  • Manual hand-guiding│        │  • Gripper dual-joint   │    │
│   └──────────────────────┘        └─────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
reBot-Isaacsim/
├── pyproject.toml                           # uv workspace configuration
├── README.md
├── README_EN.md
├── README_ES.md
├── reBotArm_Isaacsim/                       # Main example directory
│   ├── gravity_joint_sender.py              # Real-robot sender (uv environment)
│   ├── isaacsim_ik_sender.py                # IK sender (must use Isaac python.sh)
│   ├── isaacsim_traj_sender.py              # Trajectory sender (must use Isaac python.sh)
│   ├── isaacsim_joint_test_sender.py        # Test sender (use python.sh when needed)
│   ├── joint_reader_sender.py               # Read-only mapping script
│   ├── isaacsim_joint_receiver.py           # Isaac Sim receiver (must use Isaac python.sh)
│   ├── live_sync.py                         # Startup guide script
│   └── ...
├── third_party/
│   └── reBotArm_control_py/                 # Robot control library (independent uv environment)
│       ├── pyproject.toml
│       └── ...
├── urdf/
│   └── ...                                  # Robot URDF / configuration
├── usd/
│   └── reBot_B601_DM/
│       └── reBot_B601_DM.usda               # reBotDM asset
└── ...
```

> This repository no longer keeps wrapper scripts such as `run_sender.sh` / `run_isaacsim_receiver.sh`; please run the corresponding Python scripts directly and choose the startup method described below.

## Dependencies and prerequisites

| Component | Requirement |
|------|------|
| Robot model | Current code only supports reBotDM |
| Isaac Sim | 6.0.0; recommended to use the official zip package and extract it locally |
| Isaac Sim path | Recommended to set `ISAACSIM_ROOT` to the official installation directory |
| Interface | Default is `ttyACM0`, which can be modified in `third_party/reBotArm_control_py/config/rebotarm_dm.yaml` |
| Python | For the robot sender, Python 3.10+ managed by `uv`; Isaac Sim runtime is provided by `python.sh` |
| uv | Recommended for managing current project and `third_party/reBotArm_control_py` dependencies |
| reBotArm_control_py | `uv sync` has already been run inside `third_party/reBotArm_control_py` |

### Recommended Isaac Sim installation method

This project recommends using the official Isaac Sim zip release and extracting it to a fixed directory, for example:

```bash
# Example: extract to a common directory
mkdir -p /home/seeed/IsaacSim
# Extract the official zip into this directory
# After extraction, you should get something like:
# /home/seeed/IsaacSim/python.sh
```

### Check the USB2CAN port

```bash
# View the USB2CAN serial port to ensure the port is detected
ls ttyACM*

# Grant port permissions
sudo chmod 666 /dev/ttyACM*
```

## Environment setup

### 1. Install Isaac Sim 6.0.0

Please use the official zip package and do not rely on an incomplete Python environment or mix other Isaac Sim runtimes.
Official links and resources:
https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/quick-install.html
https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/download.html#isaac-sim-latest-release

Run the following commands to install and set environment variables:

```bash
mkdir ~/isaacsim
cd ~/Downloads
unzip "isaac-sim-standalone-6.0.0-linux-x86_64.zip" -d ~/isaacsim
cd ~/isaacsim
./post_install.sh
./isaac-sim.sh
```

### 2. reBotArm_control_py environment

```bash
cd third_party/reBotArm_control_py
uv sync
```

## Startup (dual-terminal mode)

Two separate terminals are required. **Terminal 1 is the Isaac Sim receiver, and Terminal 2 is the real robot sender**.
Local execution is possible by setting both `DEFAULT_SIM_HOST` and `DEFAULT_REBOT_ARM_HOST` to `127.0.0.1`.

### Terminal 1 — Start the Isaac Sim receiver

All scripts related to Isaac Sim must be launched directly using the official `python.sh`:

```bash
"your_isaacsim_folder_path"/python.sh  "your_workspace"/reBot-Isaacsim/reBotArm_Isaacsim/isaacsim_joint_receiver.py
```

**Expected output:**
- Launch the Isaac Sim GUI
- Load the ground and reBotDM USD assets
- Listen on UDP `DEFAULT_SIM_HOST:5005`
- Wait for the sender to connect

### Terminal 2 — Real robot sender

**Startup order: receiver first, then sender.**

```bash
cd /path/to/reBot-Isaacsim
uv run python reBotArm_Isaacsim/gravity_joint_sender.py
```

**Expected behavior:**
- Connect to the real robot and enable MIT + gravity feed-forward compensation
- The robot can be manually pushed and moved
- Joint angles are continuously transmitted over UDP at 60 Hz

#### Other Isaac Sim-related scripts

If you are running scripts related to `isaacsim`, always launch them with the official `python.sh`, for example:

```bash
export ISAACSIM_ROOT="path/to/your/isaacsim/folder"   # e.g. /home/seeed/IsaacSim/
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_traj_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_test_sender.py
```

Notes:
- `gravity_joint_sender.py` / `joint_reader_sender.py` and similar scripts that connect to the real robot or UDP packets are usually run in the current project `uv` environment.
- `isaacsim_*` scripts must use Isaac's official `python.sh`; otherwise `SimulationApp` / the Isaac Sim Python runtime may not be available.

#### Input format examples (IK / Traj sender)

**Input format (one command per line):**
```
x y z                       # position (m), keep the current orientation
x y z r p y                 # position + orientation (m/deg)
q j1 j2 j3 j4 j5 j6         # directly send joint angles (deg)
gripper <0~1>                # update the gripper only
```

**Trajectory mode input:**
```
x y z                       # position (m)
x y z r p y                 # position + orientation (m/deg)
q j1 j2 j3 j4 j5 j6         # direct joint-space command (deg)
gripper <0~1>                # update the gripper only
speed <scale>                # adjust trajectory duration ratio
resync                       # re-read the current joint angles from the simulation side
```

#### Read-only mapping mode (`joint_reader_sender`)

Read joint angles and map them to Isaac Sim. This is useful when the real robot is running other tasks while you want to visualize the motion in Isaac Sim.

```bash
cd /path/to/reBot-Isaacsim
uv run python reBotArm_Isaacsim/joint_reader_sender.py
```

**Expected behavior:**
- Reads joint angles in passive feedback mode only, without sending any control commands
- Joint angles are continuously transmitted over UDP at 60 Hz
- When the real robot is controlled by another project, its motion can also be visualized in Isaac Sim

## Communication protocol

UDP JSON on port `127.0.0.1:5005`.

**Payload sent by the sender for each frame:**

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
| `sequence` | int | Incrementing sequence number |
| `timestamp` | float | Unix timestamp (seconds) |
| `joint_positions` | float[6] | First 6 joint angles (rad) |
| `gripper_position` | float | Target gripper finger position (m); each sender uses its own conversion method (see table below) |

**Gripper control chain:**
The receiver takes the received `gripper_position` directly as the target position for the left and right sliding joints, and clips it to `[0, upper limit]` for each finger (USD limit: both fingers are 0.05 m; both are driven by the same motor through a single gear, so travel is strictly 1:1). The receiver does not apply extra scaling. The conversion from each sender to `gripper_position` is as follows:

| Sender | Conversion to `gripper_position` (m) |
|------|------|
| `gravity_joint_sender` | `gripper_q × 0.03` (`GRIPPER_POSITION_SCALE = 0.03`) |
| `joint_reader_sender` | `gripper_q × 0.007` (`GRIPPER_POSITION_SCALE = 0.007`) |
| `isaacsim_traj_sender` | `ratio × 0.045` (`gripper <0~1>` input, clipped to 0.045 m) |
| `isaacsim_ik_sender` | Raw `ratio ∈ [0, 1]` sent directly in meters, so when the ratio reaches or exceeds the finger limit, that finger is fully open |

## Configuration parameters

### Sender (`gravity_joint_sender.py`)

| Parameter | Default | Description |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | Number of joints |
| `DEFAULT_PORT` | 5005 | UDP port |
| `DEFAULT_SEND_HZ` | 60.0 | Send frequency (Hz) |
| `GRIPPER_POSITION_SCALE` | 0.03 | Scaling factor from gripper angle to position |
| `position_alpha` | 0.2 | Low-pass filter coefficient |

### Receiver (`isaacsim_joint_receiver.py`)

| Parameter | Default | Description |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | Number of joints |
| `DEFAULT_PORT` | 5005 | UDP port |
| `DEFAULT_RENDER_HZ` | 120.0 | Simulation rendering frequency (Hz) |
| `ROBOT_PRIM_PATH` | `/World/reBotArm` | Robot prim path in Isaac Sim |
| `ASSET_RELATIVE_PATH` | `usd/RS-rebot-dev-arm/RS-rebot-dev-arm.usda` | Relative USD asset path |

## Troubleshooting

### `OSError: [Errno 98] Address already in use`

Port 5005 is already in use. Check which process is occupying it and stop it:

```bash
# View which process is using the port
sudo lsof -i :5005

# Terminate it (replace PID with the actual value)
kill <PID>
```

### Isaac Sim asset not found

Confirm that the USD asset exists or check whether `REPO_ROOT` is correct:

```bash
ls usd/reBot_B601_DM/reBot_B601_DM.usda
```

### CAN bus not ready

Ensure the CAN interface is up and the bitrate is correct:

```bash
can_restart can0
# verify:
ip -details link show can0 | grep bitrate
```

### Joint angles out of sync

- Confirm that the sender and receiver use the same port (5005)
- Check whether the sender log keeps printing `[send]`
- Check whether the receiver log keeps printing `[recv]`
- Try `isaacsim_joint_test_sender.py` to rule out hardware issues

## Components and Python environments

| Component | Python environment | Launch method |
|------|------------|---------|
| Real robot sender | Current repo `uv` environment + `reBotArm_control_py` | `uv run python reBotArm_Isaacsim/gravity_joint_sender.py` |
| Read-only mapping sender | Current repo `uv` environment | `uv run python reBotArm_Isaacsim/joint_reader_sender.py` |
| Isaac Sim receiver | Official Isaac Sim Python (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_receiver.py` |
| Isaac Sim-related scripts | Official Isaac Sim Python (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py` |
