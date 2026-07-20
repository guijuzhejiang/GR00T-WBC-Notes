# 评估指南

## 训练前回放验证

训练前建议先在仿真器里回放动作数据，确认位置重定向和坐标系转换是否正确。

回放命令示例：
```bash
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    ++replay=True \
    num_envs=4 \
    headless=False
```

### 命令理解与数据路径说明：
* **默认数据路径**：该命令中并没有直接通过命令行指定数据路径，这是因为系统会自动去读取实验配置文件 `+exp=...` 中的配置。在此命令中指向的是 `gear_sonic/config/exp/manager/universal_token/all_modes/sonic_release.yaml`，其内部缺省配置了数据目录：
  - 机器人运动库路径：`motion_file: data/motion_lib_bones_seed/robot_filtered`
  - 参数化人体运动路径：`smpl_motion_file: data/bones_seed_smpl`
* **回放自定义数据**：如果是自己新处理好的动作数据，或是存放在非默认路径下的动作数据，可以通过命令行 `++` 语法将其参数覆盖。例如：
  ```bash
  python gear_sonic/train_agent_trl.py \
      +exp=manager/universal_token/all_modes/sonic_release \
      ++replay=True \
      num_envs=4 \
      headless=False \
      ++manager_env.commands.motion.motion_lib_cfg.motion_file=your/custom/robot_filtered \
      ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=your/custom/smpl_filtered
  ```
* **现象说明**：当命令加了 `++replay=True` 后，策略不会更新权重，而是在 Isaac Lab 仿真界面中逐帧地回放上述指定目录中的 PKL 动作数据线索。若看到仿真里的机器人流畅、不摔落且精准地跟着人体的参考动作动，则表示数据质量及前期的动作过滤（`filter_and_copy_bones_data.py`）环节均通过。


## 评估指标（成功率、MPJPE）

```bash
python gear_sonic/eval_agent_trl.py \
    +checkpoint=<path_to_checkpoint.pt> \
    +headless=True \
    ++eval_callbacks=im_eval \
    ++run_eval_loop=False \
    ++num_envs=128 \
    "+manager_env/terminations=tracking/eval" \
    "++manager_env.commands.motion.motion_lib_cfg.max_unique_motions=512"
```

| 参数 | 说明 |
|------|------|
| `+checkpoint` | checkpoint 路径 |
| `++eval_callbacks=im_eval` | 启用运动模仿评估回调 |
| `++run_eval_loop=False` | 跑完一轮就退出，不循环 |
| `num_envs=128` | 并行评估环境数，越多越快 |
| `"+manager_env/terminations=tracking/eval"` | 使用 eval 专用的终止条件（比训练更严格） |
| `max_unique_motions=512` | 评估用的不重复动作数量 |

**收敛标准：**

| 指标 | 收敛目标 | 说明 |
|------|----------|------|
| `success_rate` | > 0.97 | 未提前终止的 episode 比例 |
| `mpjpe_l` | < 30 mm | 局部每关节位置误差 |
| `mpjpe_g` | < 200 mm | 全局每关节位置误差 |

训练好的模型在 100K iteration 后能达到 >0.98 成功率，mpjpe_l < 29 mm。

## 渲染视频

```bash
python gear_sonic/eval_agent_trl.py \
    +checkpoint=<path_to_checkpoint.pt> \
    +headless=True \
    ++eval_callbacks=im_eval \
    ++run_eval_loop=False \
    ++num_envs=8 \
    ++manager_env.config.render_results=True \
    "++manager_env.config.save_rendering_dir=./renders" \
    ++manager_env.config.env_spacing=10.0 \
    "~manager_env/recorders=empty" \
    "+manager_env/recorders=render"
```

| 参数 | 说明 |
|------|------|
| `render_results=True` | 开启渲染 |
| `save_rendering_dir` | 视频输出目录 |
| `env_spacing=10.0` | 多环境间距，防止画面重叠 |
| `~manager_env/recorders=empty` | 取消默认的空 recorder |
| `+manager_env/recorders=render` | 换成渲染 recorder |

视频输出为 `000000.mp4`, `000001.mp4` ... 保存在 `save_rendering_dir` 里。

## 评估官方发布 checkpoint 的注意事项

官方 checkpoint 的 `config.yaml` 里包含内部训练路径，需要手动覆盖运动数据路径：

```bash
# 评估指标
python gear_sonic/eval_agent_trl.py \
    +checkpoint=sonic_release/last.pt \
    +headless=True \
    ++eval_callbacks=im_eval \
    ++run_eval_loop=False \
    ++num_envs=128 \
    "+manager_env/terminations=tracking/eval" \
    "++manager_env.commands.motion.motion_lib_cfg.max_unique_motions=512" \
    "++manager_env.commands.motion.motion_lib_cfg.motion_file=data/motion_lib_bones_seed/robot_filtered"
```

自己训练的 checkpoint（用 `sonic_release` 配置）不需要这个额外的路径覆盖。

## 快速评估流程

1. 回放数据确认 → `++replay=True`
2. 少量环境快速看效果 → `num_envs=8 headless=False`  
3. 批量评估指标 → `num_envs=128 ++eval_callbacks=im_eval`
4. 渲染视频存档 → `++manager_env.config.render_results=True`
