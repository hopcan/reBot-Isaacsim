# reBot-Isaacsim

简体中文 | [English](./README_EN.md) | [Español](./README_ES.md)

reBot-Isaacsim 是一个针对 reBotDM 机型的 NVIDIA Isaac Sim 仿真项目。当前代码仅兼容 reBotDM，不适用于其他 reBotArm / RS 系列机型。

> 重要说明：
> - 当前仅兼容 reBotDM
> - 建议使用 Isaac Sim 6.0.0
> - 推荐安装方式为下载官方 zip 压缩包并解压安装
> - 所有与 Isaac Sim 直接相关的脚本都必须通过官方 `python.sh` 启动
> - 真实机械臂发送端仍使用当前仓库的 `uv` 环境

## 功能组件概览

本项目提供多种发送端，以满足不同的使用场景：

| 组件 | 说明 |
|------|------|
| `gravity_joint_sender` | **重力补偿手柄模式**：改装机械臂（拆卸夹爪，加装手柄），通过重力补偿模式允许手动掰动，实时同步关节角到 Isaac Sim |
| `isaacsim_ik_sender` | **逆运动学（IK）模式**：输入末端位姿，通过 IK 求解器得到关节角，发送到 Isaac Sim |
| `isaacsim_traj_sender` | **轨迹规划（Traj）模式**：在 IK 基础上增加关节空间轨迹规划（MIN_JERK 时间剖面），实现平滑运动控制 |
| `isaacsim_joint_test_sender` | **关节测试模式**：无需真实机械臂，发送预设关节角轨迹，用于验证 Isaac Sim 接收端和通讯是否正常 |
| `joint_reader_sender` | **Real-to-Sim 映射模式**：只读关节角并映射到 Isaac Sim，适合与其他控制项目配合使用（例如：实际机械臂在运行其他任务时，同步映射到 Isaac Sim 进行可视化） |

## 系统架构

```
┌──────────────────────────────────────────────────────────────────┐
│                         reBot-Isaacsim                           │
│                                                                  │
│   ┌──────────────────────┐        ┌─────────────────────────┐    │
│   │ 发送端 (Terminal 2)   │  UDP   │   接收端 (Terminal 1)    │    │
│   │                      │  JSON  │                         │    │
│   │ gravity_joint_sender │──────▶ │ isaacsim_joint_receiver │    │
│   │                      │ 5005   │                         │    │
│   │  • reBotArm_control  │        │  • Isaac Sim 仿真        │    │
│   │    _py uv 环境        │        │  • 地面 + 机械臂 USD      │   │
│   │  • MIT + 重力前馈     │        │  • 关节角同步             │    │
│   │  • 允许手动掰动        │        │  • 夹爪双关节联动         │    │
│   └──────────────────────┘        └─────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

## 目录结构

```
reBot-Isaacsim/
├── pyproject.toml                           # uv 工作空间配置
├── README.md
├── README_EN.md
├── README_ES.md
├── reBotArm_Isaacsim/                       # 主示例目录
│   ├── gravity_joint_sender.py              # 真实机械臂发送端（uv 环境）
│   ├── isaacsim_ik_sender.py                # IK 发送脚本（必须用 Isaac python.sh）
│   ├── isaacsim_traj_sender.py              # 轨迹发送脚本（必须用 Isaac python.sh）
│   ├── isaacsim_joint_test_sender.py        # 测试发送脚本（视情况使用 python.sh）
│   ├── joint_reader_sender.py               # 只读映射脚本（真实机械臂/其他项目）
│   ├── isaacsim_joint_receiver.py           # Isaac Sim 接收端（必须用 Isaac python.sh）
│   ├── live_sync.py                         # 启动说明脚本
│   └── ...
├── third_party/
│   └── reBotArm_control_py/                 # 机械臂控制库（独立 uv 环境）
│       ├── pyproject.toml
│       └── ...
├── urdf/
│   └── ...                                  # 机械臂 URDF / 配置
├── usd/
│   └── reBot_B601_DM/
│       └── reBot_B601_DM.usda               # reBotDM 资产
└── ...
```

> 当前仓库中不再保留 `run_sender.sh` / `run_isaacsim_receiver.sh` 这类包装脚本；请直接运行对应的 Python 脚本，并按以下方式选择启动方式。

## 依赖与前提条件

| 组件 | 要求 |
|------|------|
| 机械臂型号 | 当前代码仅兼容 reBotDM |
| Isaac Sim | 6.0.0，建议直接下载官方 zip 安装包解压使用 |
| Isaac Sim 路径 | 推荐设置 `ISAACSIM_ROOT`，指向官方安装目录 |
| 接口 | 默认为ttyACM0，在 third_party/reBotArm_control_py/config/rebotarm_dm.yaml 修改|
| Python | 运行机械臂发送端时可用 Python 3.10+，受 `uv` 管理；Isaac Sim 自带运行时由 `python.sh` 提供 |
| uv | 推荐使用 uv 管理当前项目和 `third_party/reBotArm_control_py` 依赖 |
| reBotArm_control_py | 已在 `third_party/reBotArm_control_py` 中运行 `uv sync` |

### 推荐的 Isaac Sim 安装方式

当前项目推荐使用官方的 Isaac Sim zip 发行版，直接下载并解压到固定目录，例如：

```bash
# 例子：解压到常用目录
mkdir -p /home/seeed/IsaacSim
# 把官方 zip 解压到该目录中
# 之后会得到类似：
# /home/seeed/IsaacSim/python.sh
```

### 检查 CAN 接口

```bash
# 查看 USB2CAN 的串口，确保检测到端口
ls ttyACM* 

# 赋予端口权限
sudo chmod 666 /dev/ttyACM*
```

## 环境准备

### 1. 安装 Isaac Sim 6.0.0

请使用官方 zip 安装包，不要依赖不完整的 Python 环境或混用其他 Isaac Sim 运行时。
官方链接和资源：
https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/quick-install.html
https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/download.html#isaac-sim-latest-release

运行下面命令安装设置环境变量

```bash
mkdir ~/isaacsim
cd ~/Downloads
unzip "isaac-sim-standalone-6.0.0-linux-x86_64.zip" -d ~/isaacsim
cd ~/isaacsim
./post_install.sh
./isaac-sim.sh
```

### 2. reBotArm_control_py 环境

```bash
cd third_party/reBotArm_control_py
uv sync
```

## 启动（双终端模式）

需要两个独立终端。**终端 1 是 Isaac Sim 接收端，终端 2 是实机/发送端**。
本机可以同时运行，需要修改 DEFAULT_SIM_HOST 和 DEFAULT_REBOT_ARM_HOST 都为 127.0.0.1
### 终端 1 — 启动 Isaac Sim 接收端

所有与 Isaac Sim 相关的脚本都需直接使用官方 `python.sh` 启动：

```bash
"你的isaacsim文件夹路径"/python.sh  "你的工作空间"/reBot-Isaacsim/reBotArm_Isaacsim/isaacsim_joint_receiver.py
```

**预期输出：**
- 启动 Isaac Sim 图形界面
- 加载地面和 reBotDM USD 资产
- 监听 UDP `DEFAULT_SIM_HOST:5005`
- 等待发送端连接

### 终端 2 — 真实机械臂发送端

**启动顺序：先接收端，再发送端。**

```bash
cd /path/to/reBot-Isaacsim
uv run python reBotArm_Isaacsim/gravity_joint_sender.py
```

**预期行为：**
- 连接真实机械臂，启用 MIT + 重力前馈补偿
- 机械臂可自由掰动
- 关节角以 60 Hz 持续通过 UDP 发送

#### 其他 Isaac Sim 相关脚本

如果正在执行与 `isaacsim` 相关的脚本，统一使用官方 `python.sh` 启动，例如：

```bash
export ISAACSIM_ROOT="你的isaacsim文件夹所在路径"   例如：/home/seeed/IsaacSim/
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_traj_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_test_sender.py
```

说明：
- `gravity_joint_sender.py` / `joint_reader_sender.py` 等直接连接真实机械臂或 UDP 报文的脚本，通常在当前项目的 `uv` 环境中运行；
- `isaacsim_*` 这类脚本必须使用 Isaac 官方 `python.sh`，否则 `SimulationApp` / Isaac Sim Python 运行时可能不可用。

#### 输入格式示例（以 IK / Traj 发送器为例）

**输入格式（每行一条）：**
```
x y z                       # 位置 (米)，姿态保持当前
x y z r p y                 # 位置 + 姿态 (米/度)
q j1 j2 j3 j4 j5 j6         # 直接发送关节角 (度)
gripper <0~1>                # 单独更新夹爪
```

**轨迹模式输入：**
```
x y z                       # 位置 (米)
x y z r p y                 # 位置 + 姿态 (米/度)
q j1 j2 j3 j4 j5 j6         # 关节空间直发 (度)
gripper <0~1>                # 单独更新夹爪
speed <scale>                # 调整轨迹时长比例
resync                       # 重新从仿真端读取当前关节角
```

#### 只读映射模式（`joint_reader_sender`）

只读关节角并映射到 Isaac Sim，适合实际机械臂在运行其他任务时同步映射可视化：

```bash
cd /path/to/reBot-Isaacsim
uv run python reBotArm_Isaacsim/joint_reader_sender.py
```

**预期行为：**
- 仅读取关节角（被动反馈模式），不发送任何控制指令
- 关节角以 60 Hz 持续通过 UDP 发送
- 实际机械臂由其他项目控制时，可同时在 Isaac Sim 中可视化

## 通信协议

UDP JSON，端口 `127.0.0.1:5005`。

**发送端每帧 Payload：**

```json
{
  "sequence": 123,
  "timestamp": 1718000000.123,
  "joint_positions": [0.0, 0.1, 0.2, -0.1, 0.0, -0.02],
  "gripper_position": 0.05
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `sequence` | int | 递增序号 |
| `timestamp` | float | Unix 时间戳（秒） |
| `joint_positions` | float[6] | 前 6 个关节角（rad） |
| `gripper_position` | float | 夹爪指位置目标（m），各发送端有各自的换算方式（见下表） |

**夹爪控制链：**
接收端将收到的 `gripper_position` 直接作为左右两个滑动关节的位置目标，并按各指裁剪到 `[0, 上限]`（USD 上限：两指均为 0.05 m；两指由同一电机通过单个小齿轮驱动，行程严格 1:1）。接收端不做额外缩放。各发送端到 `gripper_position` 的换算如下：

| 发送端 | 到 `gripper_position`（m）的换算 |
|------|------|
| `gravity_joint_sender` | `gripper_q × 0.03`（`GRIPPER_POSITION_SCALE = 0.03`） |
| `joint_reader_sender` | `gripper_q × 0.007`（`GRIPPER_POSITION_SCALE = 0.007`） |
| `isaacsim_traj_sender` | `ratio × 0.045`（`gripper <0~1>` 输入，裁剪到 0.045 m） |
| `isaacsim_ik_sender` | 原始 `ratio ∈ [0, 1]` 直接按米发送，因此 ratio ≥ 某指上限时该指完全打开 |

## 配置参数

### 发送端 (`gravity_joint_sender.py`)

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | 关节数 |
| `DEFAULT_PORT` | 5005 | UDP 端口 |
| `DEFAULT_SEND_HZ` | 60.0 | 发送频率（Hz） |
| `GRIPPER_POSITION_SCALE` | 0.03 | 夹爪角到位置的缩放系数 |
| `position_alpha` | 0.2 | 低通滤波系数 |

### 接收端 (`isaacsim_joint_receiver.py`)

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | 关节数 |
| `DEFAULT_PORT` | 5005 | UDP 端口 |
| `DEFAULT_RENDER_HZ` | 120.0 | 仿真渲染频率（Hz） |
| `ROBOT_PRIM_PATH` | `/World/reBotArm` | Isaac Sim 中的机械臂 Prim 路径 |
| `ASSET_RELATIVE_PATH` | `usd/RS-rebot-dev-arm/RS-rebot-dev-arm.usda` | USD 资产相对路径 |

## 常见问题

### `OSError: [Errno 98] Address already in use`

端口 5005 已被占用。先确认并终止占用进程：

```bash
# 查看占用端口的进程
sudo lsof -i :5005

# 终止进程（将 PID 替换为实际值）
kill <PID>
```

### Isaac Sim 资产未找到

确认 USD 资产路径存在，或检查 `REPO_ROOT` 是否正确：

```bash
ls usd/reBot_B601_DM/reBot_B601_DM.usda
```

### CAN 总线未就绪

确保 CAN 接口 up 且 bitrate 正确：

```bash
can_restart can0
# 验证：
ip -details link show can0 | grep bitrate
```

### 关节角不同步

- 确认发送端和接收端端口一致（均为 5005）
- 检查发送端日志中 `[send]` 是否有持续输出
- 检查接收端日志中 `[recv]` 是否有持续输出
- 尝试使用 `isaacsim_joint_test_sender.py` 排除硬件问题

## 组件与 Python 环境

| 组件 | Python 环境 | 启动方式 |
|------|------------|---------|
| 真实机械臂发送端 | 当前仓库 `uv` 环境 + `reBotArm_control_py` | `uv run python reBotArm_Isaacsim/gravity_joint_sender.py` |
| 只读映射发送端 | 当前仓库 `uv` 环境 | `uv run python reBotArm_Isaacsim/joint_reader_sender.py` |
| Isaac Sim 接收端 | Isaac Sim 官方 Python (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_receiver.py` |
| Isaac Sim 相关脚本 | Isaac Sim 官方 Python (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py` |
