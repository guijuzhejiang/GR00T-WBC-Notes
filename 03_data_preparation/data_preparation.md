# 数据准备

SONIC 在 Bones-SEED 数据集上训练，包含 142K+ 条人体动捕序列，经过重定向适配 Unitree G1（29 DOF）。

## 整体流程

```
Bones-SEED CSV（120 FPS）
        │
        ▼  convert_soma_csv_to_motion_lib.py
   robot PKL（30 FPS）~142K 条
        │
        ▼  filter_and_copy_bones_data.py
   robot_filtered PKL（~130K 条，过滤掉 G1 无法完成的动作）
```

数据处理脚本不需要 Isaac Lab，可以在任何装了 `pip install -e gear_sonic/` 的机器上跑。

## Step 1：下载原始数据

从 HuggingFace 下载 G1 重定向 CSV（29 DOF，120 FPS）：
```
数据集：https://huggingface.co/datasets/bones-studio/seed
```

也可以直接把数据放到本地磁盘，比如：
```
/media/zzg/GJ_disk01/data/bones-studio/seed/g1/csv/
```

同时下载 SONIC checkpoint 和 SMPL 数据：
```bash
pip install huggingface_hub
python download_from_hf.py --training
```

这会下载：
- `sonic_release/last.pt`：用于微调的预训练 checkpoint
- `data/smpl_filtered/`：SMPL 编码器需要的数据

## Step 2：CSV 转 PKL

```bash
python gear_sonic/data_process/convert_soma_csv_to_motion_lib.py \
    --input /path/to/bones_seed/g1/csv/ \
    --output data/motion_lib_bones_seed/robot \
    --fps 30 \
    --fps_source 120 \
    --individual \
    --num_workers 16
```

| 参数 | 说明 |
|------|------|
| `--input` | 原始 CSV 目录 |
| `--output` | PKL 输出目录 |
| `--fps` | 目标帧率，训练用 30 就行 |
| `--fps_source` | 原始帧率，Bones-SEED 是 120 FPS |
| `--individual` | 每个动作单独保存一个 PKL 文件 |
| `--num_workers` | 并行处理进程数，按 CPU 核数调整 |

## Step 3：过滤不可执行动作

G1 关节角度和力矩有限制，去掉那些机器人物理上无法完成的动作片段：

```bash
python gear_sonic/data_process/filter_and_copy_bones_data.py \
    --source data/motion_lib_bones_seed/robot \
    --dest data/motion_lib_bones_seed/robot_filtered \
    --workers 16
```

过滤结果：去掉约 8.7% 的动作，保留 ~130K 条（共 142K）。

## 最终目录结构

```
<repo_root>/
├── data/
│   ├── motion_lib_bones_seed/
│   │   └── robot_filtered/    # 过滤后的 G1 运动（~130K 个 PKL）
│   └── smpl_filtered/         # SMPL 数据（HuggingFace 下载）
└── sonic_release/             # 预训练 checkpoint（HuggingFace 下载）
    └── last.pt
```

## SMPL 数据准备（可选但推荐）

SMPL 编码器提升训练质量，特别是人体姿势的泛化性。如果不用 SMPL，训练可以跑，但效果会差一些。

SMPL 数据已经包含在 HuggingFace 上，`download_from_hf.py --training` 会自动下载。

## 验证数据

训练前可以先回放动作，确认数据加载正确：

```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    ++replay=True \
    num_envs=4 \
    headless=False
```

看到机器人在仿真里跟着动作数据动，说明数据正常。

## 用 sample_data 快速测试

如果还没准备好完整数据，可以先用仓库自带的 sample_data：

```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=16 headless=True \
    ++manager_env.commands.motion.motion_lib_cfg.motion_file=sample_data/robot_filtered \
    ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=sample_data/smpl_filtered
```

sample_data 数据量小，跑起来快，适合验证环境和配置是否正常。

## 自定义数据源

如果有其他来源的动捕数据（非 Bones-SEED），需要自己做重定向，把人体骨骼映射到 G1 的关节上，然后输出成 CSV 格式，再走相同的转换流程。重定向工具推荐参考 [MotionBricks](../../motionbricks/)。
