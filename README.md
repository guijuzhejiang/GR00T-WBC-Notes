# GR00T-WholeBodyControl 中文文档

> 配套官方文档：https://nvlabs.github.io/GR00T-WholeBodyControl/

GEAR-SONIC 是 NVIDIA 开发的全身控制（Whole-Body Control）系统，基于强化学习训练策略后，ONNX 导出在 G1 机器人上以 50Hz 实时运行，支持遥操、VLA 推理、以及训练新机器人。

## 文档目录

| 文档 | 说明 |
|------|------|
| [01 系统概览](01_overview/overview.md) | 架构、模块关系、核心概念 |
| [02 遥操作指南](02_teleoperation/teleoperation.md) | PICO VR 头显全身遥操作 |
| [03 数据准备](03_data_preparation/data_preparation.md) | 运动数据下载、转换、过滤 |
| [04 训练指南](04_training/training.md) | 从头训练、微调、多卡训练 |
| [05 评估指南](05_evaluation/evaluation.md) | 指标评估、视频渲染 |
| [06 部署指南](06_deployment/deployment.md) | ONNX 导出、C++ 部署 |
| [07 训练新机器人](07_new_robot/new_robot.md) | 接入新机器人的完整流程 |
| [08 VLA 工作流程](08_vla_workflow/vla_workflow.md) | 数据采集、微调、推理全流程 |
| [09 最佳实践](09_best_practices/best_practices.md) | 常见坑、调参建议、经验总结 |
| [10 故障排除](10_troubleshooting/troubleshooting.md) | 常见报错及解决方案 |

## 快速入门

```bash
# 1. 环境检查
python check_environment.py --training

# 2. 冒烟测试（用 sample_data）
python gear_sonic/train_agent_trl.py \
    +exp=manager/universal_token/all_modes/sonic_release \
    num_envs=16 headless=False \
    ++algo.config.num_learning_iterations=5

# 3. 导出 ONNX
python gear_sonic/eval_agent_trl.py \
    +checkpoint=<path_to_checkpoint.pt> \
    +headless=True ++num_envs=1 \
    +export_onnx_only=true
```
