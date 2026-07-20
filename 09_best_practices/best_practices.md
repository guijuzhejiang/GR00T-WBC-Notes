# 最佳实践

## 训练相关

### 显存与环境数 num_envs 的关系

`num_envs` 是并行仿真环境数，直接影响训练速度和显存占用：

- 单卡 A100 80G：`num_envs=512` 是安全起点，显存够的话可以试 1024
- 4×A100：`num_envs=2048`
- 8×A100：`num_envs=4096`
- 官方用 8×RTX Pro 6000：`num_envs=4096`

扩大 `num_envs` 的好处：throughput/fps 线性提升，收敛更快。

### 从微调而非从头开始

除非有充分理由，否则总是从官方 checkpoint 微调，而不是从零开始。

从头训练需要 249 小时（8×RTX Pro 6000），微调通常几十小时就能收敛，而且效果更稳定。

```bash
# 下载官方 checkpoint
python download_from_hf.py --training

# 微调
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    +checkpoint=sonic_release/last.pt \
    num_envs=4096 headless=True \
    ...
```

### checkpoint 保存策略

默认每 2000 步保存一次，一个 checkpoint 约 4h20m（1 GPU）。`last.pt` 会频繁更新，是最新状态。

如果担心训练中断丢进度，可以缩短保存间隔：
```yaml
# gear_sonic/config/base.yaml 或命令行覆盖
++trainer.save_interval=500
```

### 训练监控指标优先级

指标太多，重点看这几个：

1. **`time_out`（episode 完成率）** > 0.90，说明机器人没频繁摔倒
2. **`tracking_vr_5point_local`** > 0.80，追踪质量
3. **`rewards/body_pos_err`** < 0.10，身体位置误差

前两个没达标时不用太在意其他指标，基础要先稳。

### 回放验证数据质量

每次换数据集，都先做一次回放检查，成本低但能避免很多问题：

```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    ++replay=True num_envs=4 headless=False
```

看到机器人在仿真里跟着数据动就说明数据路径和格式对了。

### W&B 离线模式

无外网环境：
```bash
WANDB_MODE=offline python gear_sonic/train_agent_trl.py ...
```

本地调试：
```bash
python gear_sonic/train_agent_trl.py ... use_wandb=false
```

## 数据相关

### 数据量和质量

- 过滤很重要：8.7% 的问题动作会明显影响训练稳定性
- 142K 条是官方的规模，但 50K+ 条高质量数据也能训出不错的效果
- 帧率下采样到 30 FPS 是经验值，太高增加计算量但提升有限

### SMPL 数据的价值

SMPL 编码器单独处理人体骨骼数据，用于 VLA 推理链路中的姿态估计。如果只做 G1 遥操，可以先跳过；如果打算做 VLA，SMPL 数据从一开始就要准备好。

## 遥操相关

### 校准是质量的关键

每次上机前重新校准，不要省这一步。两脚并拢、直立、手臂完全自然下垂，按完等 1-2 秒再动。

感觉有偏差随时重新校准。

### 网络延迟影响

WiFi 延迟 > 20ms 就会明显感到追踪卡顿。有条件用有线，或者让工作站尽量靠近路由器。

不要和其他高带宽设备共同用同一 WiFi 信道。

### 从仿真到真机的过渡

一定先在仿真里充分练习，特别是：
1. 模式切换（Planner ↔ Pose）时的安全程序
2. 习惯正常走路的感觉（不要过慢，不要配合机器人）
3. 紧急停止练几遍（A+B+X+Y，或者键盘 O）

## 部署相关

### ONNX 精度选择

`--policy-precision 16` 比 `32` 快，但精度略低。推荐做法：
- 先用 `32` 测试效果
- 效果满意后再切 `16` 测是否有显著退化

### 输入类型选择

| 场景 | 推荐 input-type |
|------|----------------|
| 遥操 | `zmq_manager` |
| VLA 推理 | `zmq_manager` |
| 开发调试 | `manager`（支持动态切换） |
| 键盘原型验证 | `keyboard` |

### 观测配置文件

`observation_config.yaml` 控制策略接受哪些输入特征、怎么归一化。不要随便改，除非你清楚每个字段的含义。不同 checkpoint 可能对应不同的观测配置。

## 新机器人相关

### 最容易出错的地方

1. **关节顺序**：Isaac Lab 和 MuJoCo 解析的顺序不一样，必须明确映射
2. **Body 名称**：奖励/终止配置里引用的 body 名必须在你的 URDF 里存在
3. **KP/KD 参数**：先用保守值，仿真稳定了再调

### 验证顺序

```
1. headless=False num_envs=4 回放数据 → 机器人加载正常？
2. headless=False num_envs=16 训练 100 步 → 不崩，metrics 有输出？
3. headless=True num_envs=512 训练 → 正式跑
```

每一步没问题再往下走，省得后面发现问题难定位。

## 常见陷阱

- **`++` vs `+`**：`++` 是覆盖已有值，`+` 是添加新值（如 `+checkpoint=`、`+exp=`）
- **motion_file 路径**：必须是目录（包含多个 PKL），不是单个文件
- **checkpoint 路径**：用于微调的 `+checkpoint=` 是 `.pt` 文件路径
- **头显固件**：PICO 头显和 XRoboToolKit 版本要匹配，不要随意更新
- **Too many open files**：训练入口加 `mp.set_sharing_strategy("file_system")`
