# 训练新机器人

把 SONIC 移植到新机器人，核心思路是：告诉系统你的机器人有哪些关节、怎么控制、运动数据怎么对应。一共七步，不复杂，但每步都不能漏。

## 需要准备什么

1. **机器人模型文件**：URDF/USD（Isaac Lab 仿真用）+ MJCF/XML（运动库正向运动学用）
2. **重定向后的运动数据**：人体动作重定向到你的机器人骨骼，PKL 格式
3. **机器人配置**：关节名、执行器参数、动作缩放
4. **实验配置 YAML**：把上面三项拼起来

## 需要修改的文件清单

| 文件 | 操作 | 作用 |
|------|------|------|
| `gear_sonic/data/assets/robot_description/urdf/<robot>/` | 新建 | URDF + 网格文件 |
| `gear_sonic/data/assets/robot_description/mjcf/<robot>.xml` | 新建 | MuJoCo XML |
| `gear_sonic/envs/manager_env/robots/<robot>.py` | 新建 | 机器人配置（最重要的文件） |
| `gear_sonic/envs/manager_env/robots/__init__.py` | 修改 | import 新模块 |
| `gear_sonic/envs/manager_env/modular_tracking_env_cfg.py` | 修改 | 加入 `robot_mapping` 注册 |
| `gear_sonic/trl/utils/order_converter.py` | 修改 | 添加坐标转换类 |
| `gear_sonic/config/exp/manager/universal_token/all_modes/sonic_<robot>.yaml` | 新建 | 实验配置 |
| 各 config YAML（terminations、rewards、commands） | 检查 | body 名称是否匹配 |

## Step 1：机器人模型文件

```
gear_sonic/data/assets/robot_description/
├── urdf/your_robot/
│   ├── your_robot.urdf
│   └── meshes/         # STL/OBJ 网格文件
└── mjcf/
    └── your_robot.xml  # MuJoCo XML
```

URDF 给 Isaac Lab 做物理仿真，MJCF 给运动库做正向运动学。**两个文件必须代表同一个机器人，关节名和树形结构要一致。**

URDF 网格路径用相对路径（如 `meshes/pelvis.stl`），避免用 `package://` 前缀。

## Step 2：机器人配置

新建 `gear_sonic/envs/manager_env/robots/your_robot.py`，这是最核心的文件，定义了：

- 关节名称列表（Isaac Lab 顺序 vs MuJoCo 顺序的映射）
- KP/KD 执行器参数
- Isaac Lab ArticulationCfg
- 动作缩放（action scale）
- VR 追踪用的身体名称

参照 H2 的实现 `gear_sonic/envs/manager_env/robots/h2.py` 来写，结构完全一样。

**关节顺序很重要**：Isaac Lab 和 MuJoCo 解析 URDF/XML 的关节顺序可能不同，需要明确定义两套顺序和它们之间的映射。

**KP/KD 调参建议：** 先从保守值（较小 KP，适中 KD）开始，仿真里不崩了再慢慢调大。

## Step 3：Order Converter

在 `gear_sonic/trl/utils/order_converter.py` 里添加你的机器人转换类：

```python
class YourRobotConverter(IsaacLabMuJoCoConverter):
    def __init__(self):
        from gear_sonic.envs.manager_env.robots.your_robot import (
            YOUR_ROBOT_ISAACLAB_JOINTS,
            YOUR_ROBOT_ISAACLAB_TO_MUJOCO_BODY,
            YOUR_ROBOT_ISAACLAB_TO_MUJOCO_DOF,
            YOUR_ROBOT_MUJOCO_TO_ISAACLAB_BODY,
            YOUR_ROBOT_MUJOCO_TO_ISAACLAB_DOF,
        )
        self.JOINT_NAMES = YOUR_ROBOT_ISAACLAB_JOINTS
        self.DOF_MAPPINGS = {
            ("isaaclab", "mujoco"): YOUR_ROBOT_ISAACLAB_TO_MUJOCO_DOF,
            ("mujoco", "isaaclab"): YOUR_ROBOT_MUJOCO_TO_ISAACLAB_DOF,
        }
        self.BODY_MAPPINGS = {
            ("isaaclab", "mujoco"): YOUR_ROBOT_ISAACLAB_TO_MUJOCO_BODY,
            ("mujoco", "isaaclab"): YOUR_ROBOT_MUJOCO_TO_ISAACLAB_BODY,
        }

    VR_3POINTS_BODY_NAMES = ["torso_link", "left_wrist_pitch_link", "right_wrist_pitch_link"]
    FOOT_BODY_NAMES = ["left_ankle_roll_link", "right_ankle_roll_link"]
```

**注意：** 用延迟导入（import 放在 `__init__` 内部），避免循环依赖。

## Step 4：检查 Body 名称一致性

这是最容易出错的地方。训练配置里引用了很多具体的 body 名称，必须确保这些名称在你的机器人 URDF/MJCF 里存在：

需要检查的配置文件：
- `config/manager_env/commands/terms/motion.yaml`：运动命令配置
- `config/manager_env/terminations/terms/`：终止条件（根节点、接触判定用的 body）
- `config/manager_env/rewards/terms/`：奖励函数（追踪点的 body 名称）

**如果你的机器人 body 名称和默认配置不一致，在实验 YAML 里覆盖：**

```yaml
manager_env:
  commands:
    motion:
      reward_point_body: ["torso_link", "left_wrist_roll_link", "..."]
  # 覆盖 terminations 里的 body 名称类似处理
```

## Step 5：运动数据

接入新机器人时，运动数据分为人类运动数据（`smpl_filtered`）与机器人动作数据（`robot_filtered`）。关于这两部分数据的处理与理解：

### 1. 人类运动数据（`smpl_filtered`）
* **结论**：**无需修改，可直接复用**。
* **原因**：SMPL 仅代表标准的参数化人体姿态结构，与具体的机器人几何和自由度（DOF）配置无关。只要您的实验配置文件需要使用 SMPL 辅助训练（例如为了与 VLA 的 SMPL 提取隐空间对齐），直接复用已有的 `data/smpl_filtered/` 里的 PKL 数据即可。

### 2. 机器人运动数据（`robot_filtered`）
* **结论**：**不能直接将 `g1` 目录下的原始 CSV 作为输入！**
* **重要限制说明**：HuggingFace 上的原始数据（如 `bones-studio/seed/g1/csv/`）中存储的已经是**重定向到 Unitree G1 机械结构和 DOF（29自由度）** 后的角度值和位移。不同机器人的关节拓扑（如 H2 只有 19 自由度，且关节限位不同）完全不同。

* **正确的操作链条与重定向实现方法**：
  我们提供或推荐了以下三种具体的重定向（Retargeting）开源与内置方法来实现从人体动捕数据到新机器人的转换：

  - **方法 A：NVIDIA SOMA-Retargeter（官方外部库，推荐）**
    - **说明**：这是官方将 Bones-SEED 人体动捕标记点转换为 G1 关节数据时所采用的基于优化（Newton 求解器）的重定向系统。
    - **开源仓库**：[soma-retargeter](https://github.com/NVIDIA/soma-retargeter)
    - **使用方法**：提供人类 SMPL 姿态节点或动捕 CSV，指定您的新机器人 URDF 模型，它会自动将末端执行器和身体关节做优化匹配计算，并输出可以直接被 `convert_soma_csv_to_motion_lib.py` 识别的各个关节角度序列。

  - **方法 B：内置 naive PyTorch 重定向模块（本地代码）**
    - **说明**：本项目内部的 `gear_sonic/utils/motion_lib/skeleton.py` 包含了一个简单的纯 PyTorch / Vectorized 重定向实现。
    - **具体接口**：`SkeletonState.retarget_to()` 和 `SkeletonState.retarget_to_by_tpose()`。
    - **使用方法**：
      1. 在 Python 代码中导入并创建源机器人与目标机器人的 `SkeletonTree`（读取各自 MJCF XML）。
      2. 编写 `joint_mapping: dict[str, str]` 显式建立人体关节名到你机器人关节名的映射对。
      3. 定义人类与机器人各自的 T-pose 关节旋转矩阵 `source_tpose` 与 `target_tpose`，以及尺度变化系数 `scale_to_target_skeleton`。
      4. 通过调用 `source_state.retarget_to(joint_mapping, ...)`，可在内存中直接将原始 BVH 动作等速投影计算成机器人局部的关节旋转，最后转成 CSV 格式存储。

  - **方法 C：使用子项目 MotionBricks 重定向与生成系统（本地子目录）**
    - **说明**：在本项目根目录下的 `motionbricks/` 文件夹拥有完整的运动特征处理与表示逻辑。
    - **开发指导**：参考目录下的 [adding_your_own_dataset.md](file:///home/zzg/workspace/pycharm/GR00T-WholeBodyControl/motionbricks/docs/adding_your_own_dataset.md) 和 [motion_representation.md](file:///home/zzg/workspace/pycharm/GR00T-WholeBodyControl/motionbricks/docs/motion_representation.md)。
    - **使用方法**：若您的新机器人结构与 G1 大相径庭，可在 `motionbricks/motionlib/core/motion_reps/` 中重写您的机器人 `MotionRep`，通过其中的 Forward Kinematics 求解特征，并利用 `motionbricks/scripts/train_vqvae.py` 等算法合成参考轨迹。

  1. **第一步（执行上述重定向）**：通过选定的重定向手段，生成您新机器人的关节角度 CSV 数据并保存至新文件夹（例如 `data/bones-studio/seed/your_robot/csv/`）。

  2. **第二步（修改转换脚本）**：在 `gear_sonic/data_process/convert_soma_csv_to_motion_lib.py` 中，针对您新机器人的机械结构，修改以下五个核心常量：
     - `MJ_TO_IL`：新机器人从 MuJoCo 自由度顺序到 Isaac Lab 自由度顺序的全局映射数组。
     - `NUM_DOF`：新机器人的总执行自由度（例如 G1 为 29，H2 为 19）。
     - `NUM_BODIES`：新机器人的全身主要链接（Actuated Links + Pelvis）体数量。
     - `DOF_AXIS`：每个执行关节在局部坐标下的旋转轴向（`x/y/z` 对应的单位数矩阵）。
     - `BONES_CSV_JOINT_NAMES`：新机器人的重定向 CSV 首行对应的关节列名称顺序。
  3. **第三步（执行转换与过滤）**：
     ```bash
     # 指向您重定向后新机器人的 CSV 数据源，并生成原始 robot 镜像
     python gear_sonic/data_process/convert_soma_csv_to_motion_lib.py \
         --input /path/to/your_robot/csv/ \
         --output data/motion_lib_your_robot/robot \
         --fps 30 --fps_source 120 --individual --num_workers 16
     
     # 执行过滤逻辑，过滤掉物理越限或违背重心学的动作片段（此脚本基于动作名称关键字，无需改动）
     python gear_sonic/data_process/filter_and_copy_bones_data.py \
         --source data/motion_lib_your_robot/robot \
         --dest data/motion_lib_your_robot/robot_filtered --workers 16
     ```


## Step 6：实验配置文件

复制 `sonic_release.yaml` 改：

```yaml
# @package _global_
defaults:
  - /algo: ppo_im_phc
  - /manager_env: base_env
  # ... 和 sonic_release.yaml 一样

project_name: TRL_YourRobot_Track  # 改名字

manager_env:
  config:
    robot:
      type: your_robot  # 必须和 robot_mapping 里的 key 一致
  commands:
    motion:
      motion_lib_cfg:
        motion_file: null  # 命令行传入
        asset:
          assetFileName: "your_robot.xml"  # 你的 MJCF 文件名
```

**需要检查并按需覆盖的字段：**
- `robot.type`：必须和 `robot_mapping` 里的 key 一致
- `motion_lib_cfg.asset.assetFileName`：你的 MJCF 文件名
- `reward_point_body`：奖励计算用的 body
- `vr_3point_body`：VR 遥操追踪点（做遥操才需要）
- `upper_body_augment_prefixes`：如果你的运动数据命名方式不同，可能需要调整或删除

## Step 7：注册机器人

`__init__.py` 里添加 import：
```python
from gear_sonic.envs.manager_env.robots.your_robot import YourRobotCfg
```

`modular_tracking_env_cfg.py` 的 `robot_mapping` 字典里添加（约 998 行）：
```python
robot_mapping = {
    "g1": G1Cfg,
    "h2": H2Cfg,
    "your_robot": YourRobotCfg,  # 添加这行
}
```

## Step 8：训练

先用少量环境有界面跑，确认机器人加载正确、动作播放正常：

```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_your_robot \
    num_envs=16 headless=False \
    ++manager_env.commands.motion.motion_lib_cfg.motion_file=<path/to/your_robot_motions>
```

没问题后再开大规模训练：
```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_your_robot \
    num_envs=4096 headless=True \
    ++manager_env.commands.motion.motion_lib_cfg.motion_file=<path/to/your_robot_motions>
```

## 参考实现：H2

仓库里已经包含了 H2 机器人的完整接入，可以直接抄：

| 组件 | 文件 |
|------|------|
| 机器人配置 | `gear_sonic/envs/manager_env/robots/h2.py` |
| URDF + 网格 | `gear_sonic/data/assets/robot_description/urdf/h2/` |
| MJCF | `gear_sonic/data/assets/robot_description/mjcf/h2.xml` |
| 实验配置 | `gear_sonic/config/exp/manager/universal_token/all_modes/sonic_h2.yaml` |
| Order converter | `gear_sonic/trl/utils/order_converter.py`（`H2Converter`） |

## 检查清单

接入新机器人前，逐项确认：

- [ ] URDF 和 MJCF 关节名一致，树结构一致
- [ ] `robots/your_robot.py` 中关节顺序定义正确
- [ ] KP/KD 参数合理（仿真不崩）
- [ ] OrderConverter 中 body/DOF 映射完整
- [ ] 奖励和终止配置里的 body 名能在 URDF 里找到
- [ ] 实验 YAML 里 `robot.type` 和 `robot_mapping` key 一致
- [ ] 运动数据已转换并过滤为 PKL
- [ ] `headless=False` 跑起来机器人加载正常、动作回放正常
