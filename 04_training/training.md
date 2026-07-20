# 训练指南

## 前置检查

```bash
python check_environment.py --training
```

先跑这个，确认所有依赖都装好了。

## 冒烟测试

新环境第一次训练，用小规模快速验证：

```bash
# 有界面版（可以看到仿真画面）
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=16 headless=False \
    ++algo.config.num_learning_iterations=5

# 无界面版（服务器用）
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=16 headless=True \
    ++algo.config.num_learning_iterations=5
```

初始化需要约 1 分钟，之后控制台会开始打印训练指标。

## 用 sample_data 测试

还没准备好完整数据时，用 HuggingFace 的 sample_data：

```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=16 headless=True \
    ++manager_env.commands.motion.motion_lib_cfg.motion_file=sample_data/robot_filtered \
    ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=sample_data/smpl_filtered
```

## 从头训练

```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=4096 headless=True \
    ++manager_env.commands.motion.motion_lib_cfg.motion_file=data/motion_lib_bones_seed/robot_filtered \
    ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=data/smpl_filtered
```

官方用 8×RTX Pro 6000 从头训练，跑了 68K 次 iteration，共 249 小时收敛。

## 微调（推荐起点）

基于官方发布的 checkpoint 微调，收敛快很多：

```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    +checkpoint=sonic_release/last.pt \
    num_envs=4096 headless=True \
    ++manager_env.commands.motion.motion_lib_cfg.motion_file=data/motion_lib_bones_seed/robot_filtered \
    ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=data/smpl_filtered
```

`+checkpoint` 加号语法表示加载已有权重继续训练，不会覆盖其他配置。

## 多卡训练

### 单节点 8 GPU

```bash
accelerate launch --num_processes=8 gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=4096 headless=True
```

显存够的情况下可以把 `num_envs` 扩大 4-20 倍。

### 多节点分布式

```bash
accelerate launch \
    --multi_gpu \
    --num_machines=8 \
    --num_processes=64 \
    --machine_rank=$MACHINE_RANK \
    --main_process_ip=$MASTER_ADDR \
    --main_process_port=$MASTER_PORT \
    gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=4096 headless=True
```

官方推荐 64+ GPU 才有合理的收敛时间，单节点 8 GPU 能跑，但很慢。

## 本地调试

本地调试设 100 步就够了，不要跑太多浪费时间：

```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=16 headless=False \
    ++algo.config.num_learning_iterations=100
```

## 常用参数说明

| 参数 | 说明 |
|------|------|
| `+exp=...` | 实验配置，决定用哪套 encoder/decoder |
| `num_envs=4096` | 并行仿真环境数，越多训练越快，但显存要够 |
| `headless=True` | 不显示仿真界面，服务器必须 True |
| `+checkpoint=<path>` | 加载 checkpoint 继续训练 |
| `++algo.config.num_learning_iterations=5` | `++` 表示覆盖 YAML 中的值 |
| `++manager_env.commands.motion.motion_lib_cfg.motion_file=<path>` | 运动数据路径 |
| `++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=<path>` | SMPL 数据路径 |

## W&B 日志

默认开启，几个常用覆盖：

```bash
# 离线模式
WANDB_MODE=offline python gear_sonic/train_agent_trl.py ...

# 自定义项目和团队
python gear_sonic/train_agent_trl.py ... \
    wandb.wandb_project=my_project \
    wandb.wandb_entity=my_team

# 完全关掉
python gear_sonic/train_agent_trl.py ... use_wandb=false
```

## 训练监控

关键指标（W&B `Episode_Reward/` 下）：

| 指标 | 收敛目标 | 含义 |
|------|----------|------|
| `tracking_vr_5point_local` | > 0.80 | 5 点追踪质量 |
| `tracking_relative_body_pos` | > 0.44 | 上身位置追踪 |
| `tracking_anchor_pos` | > 0.14 | 根节点位置追踪 |
| `time_out` | > 0.90 | episode 完成率 |

其他监控指标：

| 指标 | 良好范围 | 含义 |
|------|----------|------|
| `rewards/total` | 3.0+ | 总奖励 |
| `rewards/anchor_pos_err` | < 0.15 | 根节点位置误差（m） |
| `rewards/body_pos_err` | < 0.10 | 身体位置追踪误差（m） |
| `throughput/fps` | ~4000+ | 训练吞吐量 |

## Checkpoint 保存

默认每 2000 步保存一次，保存到：
```
logs_rl/TRL_G1_Track/<实验名>-<时间戳>/
├── model_step_002000.pt
├── last.pt        # 最新 checkpoint，每隔一段时间更新
└── config.yaml
```

每 2000 步大约需要 4h20m（单机 8 GPU），完整收敛到 68K 步需要约 150 小时。

## 查看配置

所有训练参数的默认值都在 `gear_sonic/config/` 里，命令行里用 `++` 覆盖的参数，基本都能在这里找到对应的 YAML 文件：

```
gear_sonic/config/
├── base.yaml                          # 全局基础配置
├── algo/                              # PPO 算法参数
├── manager_env/                       # 仿真环境配置
│   ├── commands/terms/motion.yaml     # 运动命令参数
│   ├── terminations/                  # 终止条件
│   └── rewards/                       # 奖励函数
└── exp/manager/universal_token/all_modes/  # 实验配置
    ├── sonic_release.yaml             # 默认实验
    └── sonic_h2.yaml                  # H2 机器人实验
```
