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

## robot_filtered 与 smpl_filtered 说明

训练 SONIC 时需要准备这两类核心运动数据。它们的物理意义和作用如下：

| 数据类型 | 物理意义 | 作用 | 包含的内容 |
| :--- | :--- | :--- | :--- |
| **`robot_filtered`** | **机器人关节空间动捕数据** | 指导机器人在仿真环境中的物理运动参考轨迹，用于训练 WBC 策略的关节执行与控制（Decoder 端）。 | G1 机器人的各个关节角度（29 DOF）、身体根节点（Pelvis）的三维位置和朝向四元数等。 |
| **`smpl_filtered`** | **参数化人体模型姿态数据** | 用于训练专门的 **SMPL 编码器**（SMPL Encoder），实现人体任意姿势到通用 Token 空间的跨模态映射。 | 人体 SMPL 模型对应的 24 个骨骼关节旋转（以轴角表示）、三维根节点位移等。 |

### 它们是「一一对应」的吗？

**是的一一对应。**

在通用 Token 架构中，系统通过对比学习（Auxiliary Losses）和联合训练，将不同的控制输入（机器人关节、VR 三点追踪、SMPL 人体姿态）投影到同一个共享隐空间中。因此，训练过程中**必须拥有配对（Aligned）的数据**。

- **文件名对齐**：对于 `robot_filtered/` 文件夹下的某一个动作文件（例如 `run_stride_01.pkl`），在 `smpl_filtered/` 文件夹下**必须**存在同名的 `run_stride_01.pkl`。
- **时间帧对齐**：两者的时间序列长度、时间戳必须精确一致，代表同一个动捕动作在“机器人空间”与“人体空间”的同步映射。

### 加载与匹配机制

代码加载逻辑位于 `gear_sonic/utils/motion_lib/motion_lib_base.py` 中：
1. 系统首先读取机器人动作目录（`robot_filtered/`），将其所有动作文件名（不含后缀）作为主索引键值列表 `_motion_data_keys`。
2. 然后，系统遍历每个键值，去 `smpl_motion_file` 指定的目录中寻找同名的 `.pkl` 文件。
3. 若找到同名文件，则在内存中将其与对应的机器人参考动作绑定；若未找到，该项将被设置为 `None`（此时相关 SMPL 编码器损失无法计算，可能会影响模态对齐泛化效果）。

> [!IMPORTANT]
> 如果你要使用自定义的动捕动作进行训练，请确保提供配对重定向后的机器人 PKL 以及对应的原版 SMPL 姿态 PKL，且两者的文件名保持完全一致。


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
