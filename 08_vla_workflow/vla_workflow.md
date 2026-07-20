# VLA 工作流程

VLA（视觉-语言-动作）工作流是在 SONIC 全身控制基础上，叠加语言指令和视觉感知能力。整体链路分四个阶段：数据采集 → VLA 微调 → 部署 → 推理。

## 整体架构

```
┌──────────────────────┐
│   Isaac-GR00T         │    VLA 模型（GPU 机器）
│   PolicyServer        │    加载微调后模型，ZMQ 服务
└──────┬───────────────┘
       │ ZMQ REQ/REP
       ▼
┌─────────────────────┐   ZMQ TCP  ┌──────────────────────┐
│  VLA Inference       │ ◄────────── │  Camera Server       │
│  run_vla_inference.py│            │  (机器人本地，OAK 相机)│
└────┬───────────┬────┘            └──────────────────────┘
     │           │
     │ ZMQ PUB   │ ZMQ SUB
     │ (动作)     │ (状态)
     ▼           ▼
┌─────────────────────┐
│  C++ Deploy          │    SONIC 全身控制器，50Hz
│  gear_sonic_deploy   │
└─────────────────────┘
```

**动作空间（G1 Sonic embodiment）：**  
78 维 = 64 维运动 token + 7 维左手关节 + 7 维右手关节

## 阶段一：数据采集

数据采集走遥操作模式，用人操控机器人完成任务，同时录制机器人状态和相机画面。

### 相机服务器（机器人端）

在机器人上部署相机服务，提供 ZMQ 图像流：

```bash
# 在机器人上克隆仓库，然后运行安装脚本
bash install_scripts/install_camera_server.sh
```

安装完成后相机服务会注册为 systemd 服务（端口 5555），开机自启。

### 启动数据采集

推荐用 tmux 一键启动（会自动开 4 个窗格）：

```bash
# 仿真模式
python gear_sonic/scripts/launch_data_collection.py --sim

# 真机（相机服务器 IP 192.168.123.164）
python gear_sonic/scripts/launch_data_collection.py \
    --camera-host 192.168.123.164 \
    --task-prompt "pick up the cup"

# 带腕部相机
python gear_sonic/scripts/launch_data_collection.py \
    --camera-host 192.168.123.164 \
    --task-prompt "pick up the cup" \
    --record-wrist-cameras
```

| 参数 | 说明 |
|------|------|
| `--task-prompt` | 任务语言描述，会录入数据集 |
| `--dataset-name` | 数据集名称，不填自动按时间戳命名 |
| `--camera-host` | 相机服务器地址，仿真用 localhost |
| `--data-exporter-frequency` | 录制频率，默认 50 Hz |
| `--record-wrist-cameras` | 同时录制腕部相机视图 |
| `--no-data-exporter` | 不录制数据，只运行遥操 |

tmux 会话管理：
- `Ctrl+b` + 方向键：切换窗格
- `Ctrl+b` + `d`：保持运行，断开 tmux
- `tmux attach -t sonic_data_collection`：重新连接

### 手动多终端方式

如果不用 tmux：

```bash
# 终端 1：C++ 部署（遥操输入）
cd gear_sonic_deploy
bash deploy.sh --input-type zmq_manager real

# 终端 2：PICO 追踪服务
source .venv_teleop/bin/activate
python gear_sonic/scripts/pico_manager_thread_server.py --manager

# 终端 3：数据录制
source .venv_data_collection/bin/activate
python gear_sonic/scripts/run_data_exporter.py \
    --task-prompt "pick up the apple" \
    --camera-host 192.168.123.164
```

### 录制控制

| 按键 | 功能 |
|------|------|
| `s` | 开始录制 episode |
| `e` | 结束并保存当前 episode |
| `d` | 丢弃当前 episode |

### 数据输出格式

数据保存为 LeRobot 格式（parquet + mp4）：
- 关节位置、速度、IMU 四元数（来自 C++）
- SMPL 身体参数（来自 PICO）
- 相机图像（来自相机服务）

### 数据后处理

```bash
source .venv_data_collection/bin/activate

# 删除已标记废弃的 episode
python gear_sonic/scripts/process_dataset.py \
    --input dataset/ \
    --remove-discarded

# 删除陈旧的 SMPL 帧（PICO 延迟导致的异常帧）
python gear_sonic/scripts/process_dataset.py \
    --input dataset/ \
    --remove-stale-smpl

# 合并多次采集的数据集
python gear_sonic/scripts/process_dataset.py \
    --merge dataset_1/ dataset_2/ dataset_3/ \
    --output merged_dataset/
```

## 阶段二：VLA 微调

用采集的数据微调 Isaac-GR00T 模型（在 [Isaac-GR00T 仓库](https://github.com/NVIDIA/Isaac-GR00T) 里操作）：

```bash
# 在 Isaac-GR00T 仓库
uv run python gr00t/train/finetune.py \
    --dataset-path /path/to/merged_dataset \
    --model-path nvidia/GR00T-N1-5-3B \
    --embodiment-tag UNITREE_G1_SONIC \
    --output-dir /path/to/finetuned_model \
    --batch-size 32 \
    --num-epochs 50
```

微调完成后，`/path/to/finetuned_model` 下有训练好的权重。

## 阶段三：部署准备

### 启动 PolicyServer（GPU 机器）

```bash
# 在 Isaac-GR00T 仓库
uv run python gr00t/eval/run_gr00t_server.py \
    --model-path /path/to/finetuned_model \
    --embodiment-tag UNITREE_G1_SONIC \
    --device cuda:0 \
    --port 5550
```

PolicyServer 加载 VLA 模型，通过 ZMQ REQ/REP 接收推理请求。

### 相机服务器（机器人端）

相机服务器在数据采集阶段已经装好了，如果没装：

```bash
bash install_scripts/install_camera_server.sh
```

## 阶段四：VLA 推理

### 一键 tmux 启动（推荐）

```bash
# 真机
python gear_sonic/scripts/launch_inference.py \
    --prompt "pick up the apple" \
    --camera-host 192.168.123.164

# 仿真
python gear_sonic/scripts/launch_inference.py --sim \
    --prompt "pick up the apple"

# 远程 PolicyServer
python gear_sonic/scripts/launch_inference.py \
    --prompt "pick up the apple" \
    --camera-host 192.168.123.164 \
    --policy-server-host 10.0.0.100 \
    --policy-server-port 5550
```

tmux 会自动创建 4 个窗格：
- Pane 0：C++ Deploy（全身控制器）
- Pane 1：键盘发布器（在这里打字控制）
- Pane 2：VLA Inference（策略推理客户端）
- Pane 3：Data Exporter（可选，录制推理数据）

### 手动分终端方式

```bash
# 终端 1：PolicyServer（GPU 机器）
uv run python gr00t/eval/run_gr00t_server.py \
    --model-path /path/to/finetuned_model \
    --embodiment-tag UNITREE_G1_SONIC \
    --device cuda:0 \
    --port 5550

# 终端 2：C++ 部署（全身控制器）
cd gear_sonic_deploy
bash deploy.sh --input-type zmq_manager real

# 终端 3：VLA 推理客户端
source .venv_inference/bin/activate
python gear_sonic/scripts/run_vla_inference.py \
    --host <policy_server_ip> \
    --port 5550 \
    --embodiment-tag unitree_g1_sonic \
    --prompt "pick up the apple" \
    --camera-host 192.168.123.164

# 终端 4：数据录制（可选）
source .venv_data_collection/bin/activate
python gear_sonic/scripts/run_data_exporter.py \
    --task-prompt "pick up the apple" \
    --camera-host 192.168.123.164
```

### 键盘控制

在 Pane 1（键盘发布器）里：

| 按键 | 功能 |
|------|------|
| 方向键 / WASD | 移动控制 |
| Tab | 切换遥操/自主模式 |
| Esc / Q | 停止 |

### 低延迟模式

对时延敏感的任务，用低延迟 checkpoint：

```bash
python gear_sonic/scripts/launch_inference.py \
    --deploy-checkpoint policy/low_latency/model \
    --deploy-obs-config policy/low_latency/observation_config.yaml \
    --camera-host 192.168.123.164 \
    --prompt "pick up the apple"
```

## 自定义初始姿势

VLA 推理启动时机器人会先过渡到初始姿势。如果任务需要特定起始姿态（如坐姿），要修改初始 token：

适合用固定 token（而非自然站立）的场景：
- 机器人需要从坐姿开始
- 需要特定手臂姿态开始

修改位置：`gear_sonic/scripts/run_vla_inference.py` 中的 `initial_token` 参数。

获取合适 token 的方法：用遥操做出目标初始姿势，从日志里提取对应的 token 值。

## 延迟补偿

如果 PolicyServer 在远端 GPU 机器，推理延迟会影响效果。常见优化：
- 有线网络优先（延迟更低稳定）
- PolicyServer 和机器人控制在同一局域网
- `--planner-precision 16` 降低精度换速度
