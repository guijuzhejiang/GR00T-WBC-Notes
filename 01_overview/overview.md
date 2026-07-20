# 系统概览

## GEAR-SONIC 是什么

GEAR-SONIC 是 NVIDIA 开发的人形机器人全身控制策略，基于模仿学习（PPO + 运动模仿），让 Unitree G1（29 DOF）能够实时追踪人体动作。

**核心特点：**
- 通用 token 架构：多种输入模式共享同一个解码器，切换无缝
- 多模态输入：VR 遥操、SMPL 人体模型、G1 关节轨迹、BVH 骨骼数据（可选）
- 50Hz 实时控制，ONNX 导出后在 C++ 中运行
- 支持 VLA 集成：把 VLA 的关节动作输出或latent action直接接入全身控制

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                     SONIC 策略网络                           │
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │  G1 编码器│ │Teleop编码│ │SMPL 编码 │ │ SOMA 编码器  │  │
│  │ (关节轨迹)│ │(VR 3点追踪)│ │(人体模型)│ │(BVH骨骼可选)│  │
│  └─────┬────┘ └─────┬────┘ └─────┬────┘ └──────┬───────┘  │
│        └────────────┴────────────┴──────────────┘          │
│                    FSQ 共享 token 空间                       │
│                           │                                 │
│                    ┌──────▼──────┐                          │
│                    │  解码器      │                          │
│                    │ (29 DOF 动作)│                          │
│                    └─────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

## 配置说明

| 配置名 | 编码器 | 用途 |
|--------|--------|------|
| `sonic_release` | G1、Teleop、SMPL | 默认配置，匹配官方发布的 checkpoint |
| `sonic_bones_seed` | G1、Teleop、SMPL、SOMA | 额外加了 SOMA 骨骼编码器 |

日常使用和微调，用 `sonic_release` 就够了。

## 模块与目录

```
gear_sonic/
├── train_agent_trl.py      # 训练入口
├── eval_agent_trl.py       # 评估 / ONNX 导出入口
├── config/                 # Hydra 配置（训练、环境、奖励等）
│   └── exp/manager/universal_token/all_modes/  # 实验配置
├── data_process/           # 数据转换和过滤脚本
├── envs/                   # Isaac Lab 仿真环境
│   └── manager_env/robots/ # 各机器人的配置文件
├── scripts/                # 遥操、VLA 推理、数据采集脚本
└── trl/                    # TRL 强化学习训练器

gear_sonic_deploy/          # C++ 部署代码（TensorRT/ONNX Runtime）
decoupled_wbc/              # 解耦全身控制（实验性）
motionbricks/               # 动作拼接工具
```

## 数据流

```
人体动捕数据 (Bones-SEED, BVH)
        │
        ▼
   运动重定向（适配 G1 骨骼）
        │
        ▼
   PKL 运动库（robot_filtered/）
        │
        ▼
   Isaac Lab 仿真 (PPO 训练)
        │
        ▼
   训练好的 .pt checkpoint
        │
        ▼
   ONNX 导出（encoder + decoder）
        │
        ▼
   C++ 实时部署 (50Hz)
```

## 关键依赖

- **Isaac Lab 2.3+**：仿真环境，训练必须
- **Python 3.11**：Isaac Lab 要求
- **HuggingFace TRL**：RL 训练框架
- **Hydra**：配置管理
- **W&B**：训练监控（可关闭）

数据处理脚本（`gear_sonic/data_process/`）不依赖 Isaac Lab，可以单独安装 `pip install -e gear_sonic/` 后使用。
