# 故障排除

## 训练/环境问题

### OSError: Too many open files

**原因：** 多进程共享内存时文件句柄耗尽。

**解决：** 在训练脚本入口（`gear_sonic/train_agent_trl.py` 第 22 行附近）加：

```python
import torch.multiprocessing as mp
mp.set_sharing_strategy("file_system")
```

### Isaac Lab 版本不匹配

**症状：** `import isaaclab` 失败，或各种 API 找不到。

**解决：** 确认 Isaac Lab 版本 >= 2.3 and < 3.0

```bash
python -c "import isaaclab; print(isaaclab.__version__)"
```

### 训练启动后什么都不打印

**原因：** Isaac Lab 场景构建需要约 1 分钟，这段时间是正常静默的。

**解决：** 等待 60-120 秒，之后才会开始打印 metrics。

### 训练 rewards 始终为 0 或极小

可能原因：
1. 运动数据路径错了（检查 `motion_file` 目录是否有 PKL 文件）
2. 数据路径是文件不是目录
3. 关节顺序映射错误（新机器人接入时常见）

验证：先跑回放模式 `++replay=True`，确认动作能正常播放。

### CUDA out of memory

把 `num_envs` 减半，或者降低 batch size：

```bash
python gear_sonic/train_agent_trl.py \
    ... \
    num_envs=256   # 从 4096 降到 256
```

### checkpoint 加载失败

**症状：** `RuntimeError: Error(s) in loading state_dict`

可能原因：checkpoint 和实验配置（encoder 类型）不匹配。确认 `+exp=` 指定的配置和 checkpoint 训练时用的一致。

## 评估问题

### 评估官方 checkpoint 报路径错误

官方 checkpoint 内嵌的 config.yaml 包含内部训练路径，需要覆盖：

```bash
python gear_sonic/eval_agent_trl.py \
    +checkpoint=sonic_release/last.pt \
    ... \
    "++manager_env.commands.motion.motion_lib_cfg.motion_file=data/motion_lib_bones_seed/robot_filtered"
```

### 渲染视频只有黑屏

检查：
- `--headless=True` 模式下渲染有时需要 `--no-display` 相关环境变量
- 确认 `save_rendering_dir` 路径有写权限

## 部署问题

### ONNX 推理频率不够

目标是 50Hz。频率不够时：
- 用 `--policy-precision 16` 降低精度
- 确认用了 TensorRT 加速（`gear_sonic_deploy` 默认会用）
- 减少 encoder 数（`*_decoder.onnx` 只需要匹配你的输入模式）

### C++ 部署连不上机器人

**网络接口错误：** 真机不要用 `lo`（回环），要用实际的网络接口名。

```bash
# 查看网络接口
ip addr show
# 常见：eth0, enp5s0, enP8p1s0
```

**CRC 校验失败：** 仿真模式加 `--disable-crc-check`，真机不要加。

### ZMQ 连接超时

症状：VLA 推理客户端等待 PolicyServer 响应超时。

检查：
1. PolicyServer 是否已经加载完模型（可能需要几分钟）
2. 防火墙是否放行了对应端口（默认 5550）
3. IP 地址是否正确

## 遥操问题

### 足部追踪不准

按顺序检查：
1. 足部追踪器电量（低于 20% 就容易漂移）
2. 固定是否牢固（建议绑在小腿而不是鞋面）
3. 裤子宽松、遮挡了追踪器 → 换紧身裤
4. 环境光照异常 → 换光线均匀的场所
5. 重新校准

### 机器人追踪延迟明显

- WiFi 延迟检测：`ping <机器人IP>` 应该 < 10ms
- 换有线连接
- 关闭其他高带宽应用

### 机器人在仿真里走路抖动

可能是 KP/KD 参数问题（新机器人接入）或者运动数据质量问题。先跑回放 `++replay=True` 确认数据质量，再排查参数。

## 数据采集问题

### 相机画面丢帧

- 检查相机服务状态：`systemctl status camera_server`
- 相机 USB 连接是否稳定（OAK 相机需要 USB 3.0）
- 降低 `--data-exporter-frequency`（从 50Hz 降到 30Hz）

### 数据集保存格式不对

确认在 `.venv_data_collection` 环境里运行数据处理脚本，不要混用其他 Python 环境。

```bash
source .venv_data_collection/bin/activate
```

## 新机器人接入问题

### 机器人加载后姿势异常

关节顺序映射问题。检查 `robot.py` 里定义的 `ISAACLAB_JOINTS` 和实际 URDF 里的关节顺序是否匹配。

快速调试：在仿真里打印关节名称顺序和值，和 URDF 对比。

### 奖励始终为 0，成功率为 0

常见原因：config 里的 body 名称在你的 URDF 里找不到。

逐一检查：
- `reward_point_body`
- `terminations` 里用到的关节名
- `commands` 里引用的 body 名

用这个命令直接看报错信息更明显：
```bash
python gear_sonic/train_agent_trl.py ... num_envs=4 headless=False ++algo.config.num_learning_iterations=3
```

### MJCF 正向运动学不对

确认 MJCF 和 URDF 关节定义一致。常见差异：
- 关节原点位置不一致
- 关节轴方向定义不同（x/y/z 顺序）

建议在 MuJoCo Viewer 里单独验证 MJCF 文件。
