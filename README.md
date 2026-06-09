# DRL Peg-in-Hole UR5

基于 PyBullet、Gymnasium 和 Stable-Baselines3 的 UR5 机械臂 peg-in-hole 深度强化学习项目。项目模拟一个带末端圆柱 peg 和 eye-in-hand 相机的 UR5 机械臂，通过视觉观测和连续位移动作完成插孔任务。

![simulation demo](./images/demo.gif)

<p align="center">
  <img src="./images/real_drl_ur.gif" alt="real robot demo 1">
  <img src="./images/real_drl_ur2.gif" alt="real robot demo 2">
</p>

## 项目特点

- 使用 PyBullet 搭建 UR5 + Robotiq 85 夹爪 + peg + 孔模型仿真环境。
- 使用 Gymnasium 封装为强化学习环境 `PegInHoleGymEnv`。
- 观测空间为末端相机渲染的 `100 x 100 x 1` 灰度图像。
- 动作空间为末端执行器在 X/Y/Z 三个方向上的连续小位移，范围为 `[-0.005, 0.005]`。
- 支持 Stable-Baselines3 中的 PPO、SAC、A2C，并使用 `MultiInputPolicy` 处理字典观测。
- 提供训练日志、checkpoint、reward 对比图和演示动图。

## 环境设计

核心环境定义在 [rlenv.py](rlenv.py)。

| 项目 | 配置 |
| --- | --- |
| 仿真引擎 | PyBullet GUI |
| 机器人模型 | `urdf/ur5_robotiq_85.urdf` |
| 目标孔模型 | `urdf/box.urdf`，网格为 `urdf/hole.STL` |
| 最大步数 | `500` steps / episode |
| 观测 | `Dict({"cam_image": Box(100, 100, 1, uint8)})` |
| 动作 | `Box(low=-0.005, high=0.005, shape=(3,))` |
| 成功条件 | 末端 XY 距离小于 `0.01`，Z 方向距离小于 `0.1` |
| 碰撞终止 | 未接近插入区域时，与桌面或孔模型发生接触会终止 episode |

奖励函数主要由末端执行器与目标孔的 XY 距离和 Z 距离构成：

```text
reward = 10 * (-dist_xy - 0.5 * dist_z)
```

当满足插入成功条件时，额外获得 `+100` 奖励。

## 项目结构

```text
.
├── main_rl.py                 # 训练、测试和 reward 曲线绘制入口
├── rlenv.py                   # Gymnasium 强化学习环境
├── checkpoints/               # 已保存模型 checkpoint
├── images/                    # 演示动图和 reward 对比图
├── logs/                      # Stable-Baselines3 Monitor 训练日志
├── meshes/                    # UR5 和 Robotiq 相关 mesh 资源
├── urdf/                      # 机器人、相机、孔模型 URDF
├── LICENSE
└── README.md
```

## 安装

建议使用 Python 3.9 或更高版本。

```bash
pip install pybullet gymnasium stable-baselines3 matplotlib pandas tensorboard
```

如果使用 GPU 训练，请确保 PyTorch 和 CUDA 环境已经正确安装。当前代码在 [main_rl.py](main_rl.py) 中使用 `device="cuda"`，如果没有 CUDA GPU，需要将其改为：

```python
device="cpu"
```

## 使用方法

所有主要操作都在 [main_rl.py](main_rl.py) 中完成。默认 `__main__` 部分会测试 SAC checkpoint。

### 训练模型

在 [main_rl.py](main_rl.py) 中选择算法并取消训练代码注释：

```python
agent_name = "sac"  # 可选: "sac", "ppo", "a2c"
train(agent_name=agent_name, total_timesteps=250000, save_freq=10000)
```

然后运行：

```bash
python main_rl.py
```

训练过程会：

- 创建 `logs/` 目录保存 Monitor 日志和 TensorBoard 日志。
- 每隔 `save_freq` 步在 `checkpoints/` 中保存一次模型。
- 训练结束后保存 `{agent_name}_final_model.zip`。

### 测试模型

项目已包含以下 checkpoint：

- `checkpoints/sac_model_160000_steps.zip`
- `checkpoints/ppo_model_160000_steps.zip`
- `checkpoints/a2c_model_160000_steps.zip`

在 [main_rl.py](main_rl.py) 中设置算法并调用测试函数：

```python
agent_name = "sac"
test_rl_model(agent_name)
```

测试函数会运行 `1000` 个 episode，并从以下路径加载模型：

```text
./checkpoints/{agent_name}_model_160000_steps.zip
```

### 绘制 Reward 曲线

调用：

```python
plot_reward_data()
```

该函数会读取：

- `logs/monitor_sac.csv`
- `logs/monitor_ppo.csv`
- `logs/monitor_a2c.csv`

并绘制三种算法在 `160000` step 以内的平滑 reward 曲线。

## 实验结果

项目中保存了三种算法的训练日志和 reward 对比图。

<p align="center">
  <img src="./images/reward_comparison.png" alt="reward comparison graph">
</p>

| Algorithm | Convergence Speed | Final Performance | Stability | Success Rate |
| --- | --- | --- | --- | --- |
| SAC | Fast | Excellent | Stable | 95.6% |
| PPO | Moderate | Poor | Average | 26.9% |
| A2C | Slow | Failed | Unstable | 0.0% |

从现有结果看，SAC 在该视觉插孔任务上收敛速度和最终成功率最好，PPO 能学习到部分策略但性能较弱，A2C 未能稳定完成任务。

## 注意事项

- 环境使用 `p.connect(p.GUI)`，运行时会打开 PyBullet 图形窗口。如果在无显示服务器的远程环境运行，需要改为 `p.DIRECT` 或配置图形转发。
- `main_rl.py` 当前没有命令行参数，训练、测试和绘图需要通过修改 `__main__` 中的函数调用切换。
- `test_rl_model()` 当前固定加载 `160000` step 的 checkpoint。如果训练步数或文件名不同，需要同步修改加载路径。
- `close()` 会调用 `p.disconnect()`；如果在同一 Python 进程里多次创建环境，建议显式关闭不用的环境。

## 参考

- [PyBullet](https://pybullet.org/)
- [Stable-Baselines3](https://github.com/DLR-RM/stable-baselines3)
- [Gymnasium](https://gymnasium.farama.org/)
- [ElectronicElephant/pybullet_ur5_robotiq](https://github.com/ElectronicElephant/pybullet_ur5_robotiq)
- [Catalyst.RL tutorial](https://github.com/arrival-ltd/catalyst-rl-tutorial)

## License

This project is released under the MIT License. See [LICENSE](LICENSE) for details.
