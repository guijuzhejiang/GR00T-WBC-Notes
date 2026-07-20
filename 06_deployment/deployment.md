# 部署指南

## ONNX 导出

训练完成后，把 PyTorch checkpoint 导出为 ONNX 格式，供 C++ 程序调用：

```bash
python gear_sonic/eval_agent_trl.py \
    +checkpoint=<path_to_checkpoint.pt> \
    +headless=True \
    ++num_envs=1 \
    +export_onnx_only=true
```

输出文件在 checkpoint 同级目录的 `exported/` 文件夹下：

| 文件 | 用途 |
|------|------|
| `*_smpl.onnx` | SMPL 编码器 + 解码器（SMPL 姿态输入） |
| `*_g1.onnx` | G1 编码器 + 解码器（关节轨迹输入） |
| `*_teleop.onnx` | Teleop 编码器 + 解码器（VR 追踪输入） |
| `*_encoder.onnx` | 所有编码器合并 |
| `*_decoder.onnx` | 仅解码器 |

根据你的输入模式选对应的 encoder+decoder 组合。遥操用 `teleop`，VLA 推理用 `smpl` 或 `g1`。

## C++ 部署程序

部署程序在 `gear_sonic_deploy/` 目录，用 `just` 构建和运行。

### 构建

```bash
cd gear_sonic_deploy
just build
```

### 频率测试（先跑这个）

部署到真机前，先测一下推理频率是否达标：

```bash
just run freq_test
```

### 仿真模式部署

```bash
# 先启动 MuJoCo 仿真
python gear_sonic/scripts/run_sim_loop.py

# 再启动部署程序（用 lo 回环接口）
just run g1_deploy_onnx_ref lo \
    policy/release/model_decoder.onnx \
    reference/example/ \
    --obs-config policy/release/observation_config.yaml \
    --encoder-file policy/release/model_encoder.onnx \
    --planner-file planner/target_vel/V2/planner_sonic.onnx \
    --input-type manager \
    --disable-crc-check
```

### 真机部署

```bash
just run g1_deploy_onnx_ref enP8p1s0 \
    policy/release/model_decoder.onnx \
    reference/example/ \
    --obs-config policy/release/observation_config.yaml \
    --encoder-file policy/release/model_encoder.onnx \
    --planner-file planner/target_vel/V2/planner_sonic.onnx \
    --input-type manager \
    --enable-motion-recording \
    --enable-csv-logs
```

或者用 deploy.sh 脚本简化：

```bash
# 仿真
bash deploy.sh --input-type zmq_manager sim

# 真机
bash deploy.sh --input-type zmq_manager real
```

### 部署参数说明

| 参数 | 说明 |
|------|------|
| `网络接口` | DDS 通信用的网络接口，真机用 `enP8p1s0`、`eth0` 等，仿真用 `lo` |
| `model_decoder.onnx` | 解码器 ONNX 文件路径 |
| `reference/` | 参考运动数据目录 |
| `--obs-config` | 观测空间配置 YAML，控制输入特征的归一化等 |
| `--encoder-file` | 编码器 ONNX 文件路径（token-based 策略必须） |
| `--planner-file` | 规划器 ONNX，zmq_manager、gamepad_manager 模式必须 |
| `--input-type` | 输入接口类型（见下表） |
| `--output-type` | 输出接口，`zmq`（默认）/ `ros2` / `all` |
| `--disable-crc-check` | 仿真模式关闭 CRC 校验 |
| `--enable-motion-recording` | 记录运动数据 |
| `--enable-csv-logs` | 记录 CSV 日志 |
| `--policy-precision 16\|32` | 策略浮点精度，16 更快 |
| `--planner-precision 16\|32` | 规划器浮点精度 |

**输入类型（--input-type）：**

| 类型 | 说明 |
|------|------|
| `keyboard` | 键盘直接控制 |
| `gamepad` | 游戏手柄 |
| `gamepad_manager` | 手柄 + 快速切换到 ZMQ/ROS2 |
| `zmq` | 网络运动流 |
| `zmq_manager` | 动态切换规划器和网络运动流（遥操用） |
| `manager` | 动态切换键盘、手柄、ZMQ、ROS2（功能最全） |
| `ros2` | ROS2 话题控制（需要编译 ROS2 支持） |

## 部署文件结构

```
gear_sonic_deploy/
├── deploy.sh                         # 快捷部署脚本
├── policy/
│   └── release/
│       ├── model_decoder.onnx        # 解码器
│       ├── model_encoder.onnx        # 编码器
│       └── observation_config.yaml   # 观测配置
├── planner/
│   └── target_vel/V2/
│       └── planner_sonic.onnx        # 速度规划器
└── reference/
    └── example/                      # 参考运动数据
```

## VLA 推理部署

VLA 任务（视觉-语言-动作）的部署走单独的推理链路，详见 [VLA 工作流程](../08_vla_workflow/vla_workflow.md)。
