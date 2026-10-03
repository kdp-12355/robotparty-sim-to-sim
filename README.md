# robotparty-sim-to-sim
isaac sim to mujoco sim


# RPO-Flat Sim2Sim 启动方法总结

## 一、前置准备（只需做一次）

### 1. 环境依赖
```bash
# mujoco_viewer（sim2sim 脚本需要）
/workspace/miniconda3/envs/mujoco_env/bin/pip install mujoco-python-viewer -i https://pypi.org/simple/

# robolab（ISAAC_DATA_DIR 等需要）
cd /workspace/robotor/roboparty_train-main
/workspace/miniconda3/envs/mujoco_env/bin/pip install -e ./robolab
```

### 2. 导出 policy（从训练 ckpt → TorchScript）
```bash
/workspace/miniconda3/envs/mujoco_env/bin/python /tmp/export_policy_flat.py
```

脚本内容要点：
- 网络结构：`780 → 512 → 256 → 128 → 23`，激活 `nn.ELU()`
- 从 ckpt 抽 `actor.*` 权重，加 `net.` 前缀
- `torch.jit.script` 后 `save` 到 `policy_1.pt`

导出路径：
```
/workspace/IsaacLab/logs/rsl_rl/rpo_flat/2026-09-16_13-36-32/policy_1.pt
```

### 3. xml 模型路径修正（已改过）
`sim2sim_rpo.py` 里的 `mujoco_model_path` 已改成：
```
/workspace/robotor/rpo_description/mjcf/rpo.xml          # 平地
/workspace/robotor/rpo_description/mjcf/rpo_terrain.xml  # 地形
```

---

## 二、每次启动流程

### 1. Windows PowerShell 建隧道
```powershell
ssh -L 5901:127.0.0.1:5901 root@192.168.37.20 -p 6037
```
保持窗口不关。

### 2. TightVNC Viewer 连接
- 地址：`localhost:5901`
- 密码：`666`

### 3. 进入 XFCE 桌面 → 右键 → Open Terminal Here

### 4. 在 VNC 桌面终端里执行
```bash
export XDG_RUNTIME_DIR=/tmp/runtime-root
mkdir -p /tmp/runtime-root
export DISPLAY=:1

cd /workspace/robotor/roboparty_train-main
/workspace/miniconda3/envs/mujoco_env/bin/python robolab/scripts/mujoco/sim2sim_rpo.py \
  --load_model /workspace/IsaacLab/logs/rsl_rl/rpo_flat/2026-09-16_13-36-32/policy_1.pt
```

---

## 三、关键参数位置（sim2sim_rpo.py）

| 参数 | 位置 | 说明 |
|------|------|------|
| `vx / vy / dyaw` | `class cmd` | 速度指令，0 时站立，0.1~0.5 时行走 |
| `sim_duration` | `class sim_config` | 仿真总时长（秒），10 秒约 10000 步 |
| `action_scale` | `class robot_config` | 0.25，要和训练一致 |
| `kps / kds` | `class robot_config` | PD 增益 |
| `default_pos` | `class robot_config` | 23 维初始关节角 |
| `usd2urdf` | `class robot_config` | 关节顺序映射 |
| `mujoco_model_path` | `class sim_config` | xml 路径 |
| 相机跟随 | 主循环 `viewer.render()` 前 | `viewer.cam.lookat = data.qpos[:3].copy()` |

---

## 四、改 vx 的命令

```bash
sed -i 's/^    vx = .*/    vx = 0.2/' \
  /workspace/robotor/roboparty_train-main/robolab/scripts/mujoco/sim2sim_rpo.py
```

---

## 五、跑完后查看结果

```bash
# 关节位置跟踪
eog /workspace/robotor/roboparty_train-main/joint_positions.png &

# 速度跟踪
eog /workspace/robotor/roboparty_train-main/base_velocities.png &
```

---

## 六、常见问题

| 现象 | 原因 | 处理 |
|------|------|------|
| `XDG_RUNTIME_DIR is invalid` | 没设环境变量 | 加 `export XDG_RUNTIME_DIR=/tmp/runtime-root` |
| 窗口弹不出 | 在 SSH 终端跑 | 必须在 VNC 桌面终端跑 |
| `GLFWError: Failed to open display` | `DISPLAY` 没设 | `export DISPLAY=:1` |
| `libcudnn.so.9` 找不到 | 用了 `_isaac_sim/python.sh` | 用 `mujoco_env` 的 python |
| `robolab.assets` 导入失败 | robolab 没装 | `pip install -e ./robolab` |
| `mujoco_viewer` 找不到 | 没装 | `pip install mujoco-python-viewer` |
| `rpo.xml exists: False` | 路径不对 | 改 `mujoco_model_path` 指到 `rpo_description/mjcf/` |
| 机器人不动 | `vx=0` | 改 `vx=0.2` |
| 跑得影子都没了 | `vx` 太大 | 降到 0.1~0.3，或让相机跟随 |
| 10 秒就退 | `sim_duration=10.0` | 改成 60 或 600 |

---

## 七、一键启动命令（VNC 桌面终端）

```bash
export XDG_RUNTIME_DIR=/tmp/runtime-root
mkdir -p /tmp/runtime-root
export DISPLAY=:1
cd /workspace/robotor/roboparty_train-main
/workspace/miniconda3/envs/mujoco_env/bin/python robolab/scripts/mujoco/sim2sim_rpo.py \
  --load_model /workspace/IsaacLab/logs/rsl_rl/rpo_flat/2026-09-16_13-36-32/policy_1.pt
```

---

## 八、其他任务对应脚本

| 任务 | 脚本 |
|------|------|
| RPO-Flat / Rough | `sim2sim_rpo.py` |
| RPO-AMP | `sim2sim_rpo_amp.py` |
| RPO-Interrupt | `sim2sim_rpo_interrupt.py` |
| RPO-BeyondMimic | `sim2sim_rpo_bm.py` |
| RPO-Parkour | `sim2sim_rpo_parkour.py` |
| RPO-AttnEnc | `sim2sim_rpo_attn_enc.py` |

AMP/BeyondMimic 类任务**必须先做过 `dataset_retarget.py` 关节重排**，否则动作会扭曲。
`RPO-Flat` | **平地盲走** 



 vx=0.2，重跑：
bash
sed -i 's/^    vx = .*/    vx = 0.2/' \  /workspace/robotor/roboparty_train-main/robolab/scripts/mujoco/sim2sim_rpo.pygrep -A4 "^class cmd" /workspace/robotor/roboparty_train-main/robolab/scripts/mujoco/sim2sim_rpo.py
确认 vx = 0.2 后重跑：
bash
export XDG_RUNTIME_DIR=/tmp/runtime-rootmkdir -p /tmp/runtime-rootexport DISPLAY=:1cd /workspace/robotor/roboparty_train-main/workspace/miniconda3/envs/mujoco_env/bin/python robolab/scripts/mujoco/sim2sim_rpo.py \  --load_model /workspace/IsaacLab/logs/rsl_rl/rpo_flat/2026-09-16_13-36-32/policy_1.pt



<img width="2432" height="1222" alt="image" src="https://github.com/user-attachments/assets/3836f334-8f6f-4ef0-9dd9-f4748e7ffef6" />

