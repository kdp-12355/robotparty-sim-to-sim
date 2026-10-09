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





# RPO-Rough（崎岖地形盲走）启动命令总结

## 前置条件（已完成）

- ✅ 脚本 `sim2sim_rpo.py` 已修复（第 76、313、315 行缩进/路径正确）
- ✅ `vx = 0.2` 已设（让它往前走）
- ✅ Rough policy 已导出：`rpo_rough/2026-09-17_06-51-42/exported/policy.pt`
- ✅ 地形 xml 存在：`rpo_description/mjcf/rpo_terrain.xml`

## 启动命令（逐行执行，不要一次粘贴多行）

```bash
export XDG_RUNTIME_DIR=/tmp/runtime-root
```

```bash
mkdir -p /tmp/runtime-root
```

```bash
export DISPLAY=:1
```

```bash
cd /workspace/robotor/roboparty_train-main
```

```bash
/workspace/miniconda3/envs/mujoco_env/bin/python robolab/scripts/mujoco/sim2sim_rpo.py --terrain --load_model /workspace/IsaacLab/logs/rsl_rl/rpo_rough/2026-09-17_06-51-42/exported/policy.pt
```

## 与 RPO-Flat 的区别（3 处）

| 项目 | RPO-Flat | **RPO-Rough** |
|------|----------|---------------|
| `--terrain` 参数 | ❌ 不加 | ✅ **必须加** |
| 地形 xml | `rpo.xml`（平地） | `rpo_terrain.xml`（崎岖地形） |
| policy 路径 | `rpo_flat/.../policy_1.pt` | **`rpo_rough/.../exported/policy.pt`** |
| 脚本 | `sim2sim_rpo.py` | `sim2sim_rpo.py`（同一个） |

## 关键参数位置

| 参数 | 位置 | 说明 |
|------|------|------|
| `vx / vy / dyaw` | `class cmd` | `vx=0.2` 前进；0 时站立 |
| `--terrain` | 命令行 | 切换崎岖地形 |
| `mujoco_model_path` | `class sim_config` 内 if/else | `if args.terrain` → terrain xml |
| `sim_duration` | `class sim_config` | 仿真总时长（秒） |
| `action_scale` | `class robot_config` | 0.25，要和训练一致 |
| `kps / kds` | `class robot_config` | PD 增益 |

## 常见坑

| 现象 | 原因 | 处理 |
|------|------|------|
| `IndentationError` | `sed` 误改脚本缩进 | 用 Python 脚本精确修复 313/315 行 |
| `NameError: model` | 第 76 行被覆盖 | 恢复 `model = mujoco.MjModel.from_xml_path(...)` |
| `bash: export: '-p' not a valid identifier` | 多行命令被拼成一行 | **逐行粘贴，不要一次复制多行** |
| 机器人不动 | `vx=0` | 改 `vx=0.2` |
| 加载了平地 | 忘了 `--terrain` | 加上 `--terrain` |

## 一键版（确认换行正常时可用）

```bash
export XDG_RUNTIME_DIR=/tmp/runtime-root; mkdir -p /tmp/runtime-root; export DISPLAY=:1; cd /workspace/robotor/roboparty_train-main; /workspace/miniconda3/envs/mujoco_env/bin/python robolab/scripts/mujoco/sim2sim_rpo.py --terrain --load_model /workspace/IsaacLab/logs/rsl_rl/rpo_rough/2026-09-17_06-51-42/exported/policy.pt
```

用 `;` 分隔，避免换行丢失的问题。



<img width="2210" height="1092" alt="image" src="https://github.com/user-attachments/assets/e2d15020-f957-423c-950b-dfeb6ff1ff17" />













export XDG_RUNTIME_DIR=/tmp/runtime-root
mkdir -p /tmp/runtime-root
export DISPLAY=:1
export MUJOCO_GL=glfw

cd /workspace/robotor/roboparty_train-main
/workspace/miniconda3/envs/mujoco_env/bin/python robolab/scripts/mujoco/sim2sim_rpo_interrupt.py \
  --load_model /workspace/IsaacLab/logs/rsl_rl/rpo_interrupt/2026-09-19_01-07-13/exported/policy.pt







<img width="1928" height="1282" alt="image" src="https://github.com/user-attachments/assets/c6113e52-fde2-4add-b42e-91b8049afb3e" />




cmd 类是类变量，直接改 vx = 0.3 就能让它走（不用键盘）。

改成 vx = 0.3
bash
sed -i 's/^    vx = .*/    vx = 0.3/' \
  /workspace/robotor/roboparty_train-main/robolab/scripts/mujoco/sim2sim_rpo_amp.py

sed -n '/^class cmd/,/^$/p' /workspace/robotor/roboparty_train-main/robolab/scripts/mujoco/sim2sim_rpo_amp.py
预期：

text
class cmd:
    vx = 0.3
    vy = 0.0
    dyaw = 0.0
    ...
重跑（VNC 桌面）
bash
export XDG_RUNTIME_DIR=/tmp/runtime-root
mkdir -p /tmp/runtime-root
export DISPLAY=:1
export MUJOCO_GL=glfw

cd /workspace/robotor/roboparty_train-main
/workspace/miniconda3/envs/mujoco_env/bin/python robolab/scripts/mujoco/sim2sim_rpo_amp.py \
  --load_model /workspace/IsaacLab/logs/rsl_rl/rpo_amp/2026-09-19_12-31-09/exported/policy.pt

<img width="2472" height="1336" alt="image" src="https://github.com/user-attachments/assets/60ec402f-2bf2-4fac-b19c-cebc251996b8" />







# RoboParty 机器人爬楼梯 Sim2Sim 调试步骤总结

结合你目前的 RoboParty 项目，机器人爬楼梯的目标是：加载楼梯 MuJoCo 场景，使用训练好的 RPO 策略控制机器人，先验证站立和慢速行走，再测试上楼梯能力。

## 一、整体流程

步骤 1：确认楼梯场景

检查 `rpo_stairs.xml` 和台阶几何体

步骤 2：加载 RPO 策略

使用训练好的 `policy.pt`

步骤 3：降低速度、延长仿真

尝试 vx = 0.2，仿真时长 120 秒

步骤 4：启动 MuJoCo

检查机器人能否稳定站立、慢走

步骤 5：测试爬楼梯

观察接近台阶、抬腿、上台阶及摔倒情况

## 二、具体操作步骤

### 第 1 步：确认楼梯地形

楼梯场景文件：

Bash

```
/workspace/robotor/rpo_description/mjcf/rpo_stairs.xml
```

检查场景中的楼梯几何体：

Bash

```
grep -nE 'stairs_pyramid_up|trap_up_tread|trap_down_tread|trap_platform|base_link' \
  /workspace/robotor/rpo_description/mjcf/rpo_stairs.xml | head -50
```

此前已经确认，场景不仅包含平面地面，还包含独立的台阶和平台几何体。机器人初始根部位置在 XML 中设置为 `pos="0 0 0.75"`，楼梯金字塔位于前方附近。

### 第 2 步：选择控制策略

当前项目中有两种需要区分的策略：

| 策略          | 用途     | 爬楼梯能力                |
| ----------- | ------ | -------------------- |
| `RPO-Flat`  | 平地盲行走  | 不保证能够爬楼梯             |
| `RPO-Rough` | 粗糙地形行走 | 值得优先测试，但不代表已经具备爬楼梯能力 |

此前使用的 RPO-Rough 策略路径：

Bash

```
/workspace/IsaacLab/logs/rsl_rl/rpo_rough/2026-09-17_06-51-42/exported/policy.pt
```

### 第 3 步：降低速度并延长仿真

脚本路径：

Bash

```
/workspace/robotor/roboparty_train-main/robolab/scripts/mujoco/sim2sim_rpo.py
```

先备份：

Bash

```
cd /workspace/robotor/roboparty_train-main

cp robolab/scripts/mujoco/sim2sim_rpo.py \
   robolab/scripts/mujoco/sim2sim_rpo.py.bak
```

将仿真时长从 30 秒改成 120 秒：

Bash

```
sed -i 's/sim_duration = 30\.0/sim_duration = 120.0/' \
  robolab/scripts/mujoco/sim2sim_rpo.py
```

注意： `vx = 0.2` 必须修改在脚本实际定义的命令类中。先检查：

Bash

```
grep -n -A10 -B3 'class cmd' \
  robolab/scripts/mujoco/sim2sim_rpo.py

grep -nE 'vx[[:space:]]*=|sim_duration[[:space:]]*=' \
  robolab/scripts/mujoco/sim2sim_rpo.py
```

确认 `vx` 所在位置后再修改，避免误改其他变量。`vx = 0.2` 是较低的前进速度指令，不代表机器人所有动作都会按比例减速。

### 第 4 步：启动楼梯场景

Bash

```
export XDG_RUNTIME_DIR=/tmp/runtime-root
mkdir -p /tmp/runtime-root
export DISPLAY=:1

cd /workspace/robotor/roboparty_train-main

/workspace/miniconda3/envs/mujoco_env/bin/python \
  robolab/scripts/mujoco/sim2sim_rpo.py \
  --terrain_xml /workspace/robotor/rpo_description/mjcf/rpo_stairs.xml \
  --load_model \
  /workspace/IsaacLab/logs/rsl_rl/rpo_rough/2026-09-17_06-51-42/exported/policy.pt
```

其中 `--terrain_xml` 指定楼梯场景，`--load_model` 指定策略文件。根据脚本已有的加载逻辑，显式指定的 XML 会优先于默认地形路径。

### 第 5 步：按顺序观察结果

调试检查清单

0/5 项

1. 机器人能否稳定站立

启动后检查是否立即摔倒。

2. 楼梯是否正确显示

若看不到楼梯，先检查相机视角，不要直接认定地形加载失败。

3. 机器人能否慢速前进

确认 vx 指令生效，并观察机器人是否能稳定行走。

4. 机器人能否接触并登上台阶

记录是台阶前停滞、碰撞、抬腿失败，还是登上台阶后失稳。

5. 能否持续行走

检查是否能连续通过多个台阶，而不只是偶然跨上一个台阶。

## 三、遇到问题时怎么排查

| 现象            | 优先排查                                   |
| ------------- | -------------------------------------- |
| 一启动就摔倒        | 初始姿态、关节顺序、默认关节角、策略与机器人模型是否匹配           |
| 画面看不到楼梯       | 相机视角、实际加载的 XML 路径、场景几何体                |
| 能走但到楼梯前停住     | 速度指令、策略训练地形、障碍方向与机器人行进方向               |
| 撞上台阶但抬腿失败     | 策略是否训练过类似台阶、动作空间和关节控制是否匹配              |
| 上一阶后摔倒        | 台阶高度、接触与碰撞、机身姿态和控制稳定性                  |
| 运行不到 120 秒就结束 | 查看循环退出条件、窗口状态以及是否发生异常；120 秒设置并不能防止提前退出 |

## 四、当前阶段的结论

你已经完成了楼梯 XML 的初步检查，并找到仿真时长参数 `sim_duration = 30.0`，准备调整为 120 秒和降低前进速度。

接下来最重要的是：先确认机器人能在楼梯场景中稳定站立并慢速接近台阶，再判断现有 RPO-Rough 策略是否真正具备爬楼梯能力。 如果它只能走平地或一般粗糙地形，下一步可能需要增加台阶地形训练、调整地形课程，或者专门训练楼梯通过策略。
![Uploading image.png…]()

