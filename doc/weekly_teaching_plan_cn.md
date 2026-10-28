# TUM FTM Teleoperated Driving: 一周教学课程

本文面向希望快速理解并运行 TUM FTM Teleoperated Driving 项目的学习者。目标是：

1. 理解项目的功能分层和 ROS 2 数据流。
2. 在 Docker 中完成最小仿真运行，不污染 DGX 宿主机环境。
3. 使用国内镜像源加速基础镜像、apt、ROS 2 和 rosdep 依赖下载。
4. 通过一周的阅读和实验掌握主要模块，为后续参与开发做准备。

## 当前已验证状态

当前环境已完成以下验证：

- Docker 镜像：`tod_operator_base:latest`
- 镜像 ID：`ddfb76bed044`
- 镜像大小：约 3.48 GB
- 架构：ARM64
- ROS 版本：Humble
- 项目构建：39 个包构建成功
- 最小仿真配置：
  - `vehicleID: edgar`
  - `mode: only_sim`
- 构建方式：在非 privileged、`--network none`、用户 `1000:1000` 的容器内完成
- 构建位置：`.dev_artifacts/build`、`.dev_artifacts/install`、`.dev_artifacts/log`

验证结果：

- `tod_both.launch.py` 可启动完整 operator / vehicle 仿真。
- 关闭 GUI 后，核心链路可运行。
- 核心节点数：15
- 核心话题数：75
- `/vehicle/statemachine/output/vehicle_status` 可持续输出状态。
- `/vehicle/network/data/to_operator/network_metrics` 可用于观察网络数据。
- 完整 X11 启动时，可看到 `tod_visual`、`manager`、`video_manager`、`InputDevice` 等 GUI 进程启动。
- 本地 `tod_both.launch.py` 一体仿真已修复 Manager 默认车辆 IP：
  - 默认值由 `10.0.0.10` 改为 `127.0.0.1`。
  - 下拉选项中 `127.0.0.1` 排在首位。
  - 验证时，`60100`、`60300`、`60400` 服务转发端口均连接成功，原始 `Network is unreachable` 报错消失。

当前限制：

- `tod_dev` 镜像未完成构建，原因是其 LLVM 18 stage 依赖 `apt.llvm.org`，下载速度过慢。
- 最小仿真运行不依赖 `tod_dev`，可使用 `tod_operator_base:latest`。
- `tod_input_devices` 当前会尝试读取 `build/.../src/virtual_joystick.qml`，但该 QML 文件实际安装在 `install/tod_input_devices/lib/...` 目录。该问题会导致虚拟摇杆 UI 报错，但不影响核心仿真链路。
- 构建时跳过了 `tod_dummy`、`tod_edgar_interface`、`tod_edgarautoware_interface` 三个包，因为它们依赖 Autoware / Tier4 消息包。`only_sim` 模式使用 `tod_vehicle_sim`，不需要这些接口。

## 国内镜像源

当前推荐使用以下国内镜像：

| 用途 | 镜像源 |
|---|---|
| Docker Hub 基础镜像 | `docker.1ms.run/library/ros:humble-ros-base` |
| Ubuntu ARM64 apt | `https://mirrors.ustc.edu.cn/ubuntu-ports` |
| ROS 2 apt | `https://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu` |
| rosdep / rosdistro | `https://mirrors.tuna.tsinghua.edu.cn/rosdistro` |

这些配置已经写入 `docker/dockerfile`，ROS 签名文件位于 `docker/ros-archive-keyring.gpg`。

若需要重新拉取基础镜像，可使用：

```bash
docker pull docker.1ms.run/library/ros:humble-ros-base
docker tag docker.1ms.run/library/ros:humble-ros-base ros:humble-ros-base
```

注意事项：

- `tod_operator_base` 中的 OpenVR 源码仍从 GitHub 拉取，如果网络较慢，可以先使用已构建好的镜像。
- `tod_dev` 目标依赖 `apt.llvm.org`，国内访问速度较慢。若必须构建，可后续单独处理 LLVM 源或移除可选工具链 stage。
- `docker.1ms.run` 当前可用，如不可用可替换为其他国内 Docker Hub 镜像代理。

## Docker 运行方式

### 1. 带完整 GUI 运行

适用于有 X11 桌面的 DGX 或工作站：

```bash
cd /home/my/workspace/autodrive/teleoperated_driving

docker run --rm -it \
  --user 1000:1000 \
  --shm-size=1g \
  --network none \
  -e DISPLAY="$DISPLAY" \
  -e XAUTHORITY=/tmp/.Xauthority \
  -v /tmp/.X11-unix:/tmp/.X11-unix:ro \
  -v "$XAUTHORITY:/tmp/.Xauthority:ro" \
  -v "$PWD/src:/home/tum/wsp/src" \
  -v "$PWD/config:/home/tum/wsp/config" \
  -v "$PWD/.dev_artifacts/build:/home/tum/wsp/build" \
  -v "$PWD/.dev_artifacts/install:/home/tum/wsp/install" \
  -v "$PWD/.dev_artifacts/log:/home/tum/wsp/log" \
  tod_operator_base:latest bash
```

进入容器后启动：

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch tod_launch tod_both.launch.py
```

本地一体仿真中，operator 和 vehicle 都在同一个容器内。Manager 窗口中应使用：

```text
IP Address Operator: 127.0.0.1
IP Address Vehicle:  127.0.0.1
```

当前 Manager 的车辆 IP 默认值已经是 `127.0.0.1`，可直接点击 `Connect`。`10.0.0.10` 和 `11.0.0.10` 是项目提供的路由/实车网络示例地址，不应在单容器本地仿真中默认使用。

如果 `docker entrypoint` 已自动 source，可以直接运行：

```bash
ros2 launch tod_launch tod_both.launch.py
```

### 2. 无 GUI 最小仿真

没有 X11 环境时，需要先在 `config/config/launch_setup.yaml` 中关闭两个 GUI 相关包：

```yaml
packages_to_launch:
  operator:
    tod_input_devices: False
    tod_visual: False
```

然后使用命令：

```bash
cd /home/my/workspace/autodrive/teleoperated_driving

docker run --rm -it \
  --user 1000:1000 \
  --shm-size=1g \
  --network none \
  -v "$PWD/src:/home/tum/wsp/src" \
  -v "$PWD/config:/home/tum/wsp/config" \
  -v "$PWD/.dev_artifacts/build:/home/tum/wsp/build" \
  -v "$PWD/.dev_artifacts/install:/home/tum/wsp/install" \
  -v "$PWD/.dev_artifacts/log:/home/tum/wsp/log" \
  tod_operator_base:latest bash
```

容器内启动：

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch tod_launch tod_both.launch.py
```

### 3. 重新构建镜像

如果需要重新构建镜像：

```bash
cd /home/my/workspace/autodrive/teleoperated_driving

docker build \
  -f docker/dockerfile \
  --target tod_operator_base \
  --build-arg DOCKER_USERNAME=tum \
  --build-arg DOCKER_UID=1000 \
  --build-arg DOCKER_GID=1000 \
  -t tod_operator_base:latest .
```

如需重新构建项目：

```bash
docker run --rm -it \
  --user 1000:1000 \
  --shm-size=1g \
  --network none \
  -v "$PWD/src:/home/tum/wsp/src" \
  -v "$PWD/config:/home/tum/wsp/config" \
  -v "$PWD/.dev_artifacts/build:/home/tum/wsp/build" \
  -v "$PWD/.dev_artifacts/install:/home/tum/wsp/install" \
  -v "$PWD/.dev_artifacts/log:/home/tum/wsp/log" \
  tod_operator_base:latest bash
```

容器内执行：

```bash
source /opt/ros/humble/setup.bash
colcon build \
  --packages-ignore tod_dummy tod_edgar_interface tod_edgarautoware_interface
source install/setup.bash
```

如需重新配置 rosdep：

```bash
rosdep update
rosdep install \
  --from-paths src \
  --ignore-src \
  --rosdistro humble \
  -y \
  --skip-keys "autoware_auto_vehicle_msgs autoware_auto_control_msgs autoware_auto_planning_msgs autoware_auto_perception_msgs autoware_adapi_v1_msgs tier4_external_api_msgs"
```

## 项目结构

项目采用 ROS 2 workspace 结构：

```text
teleoperated_driving/
├── README.md
├── doc/
│   ├── getting_started.md
│   ├── docker_workflow.md
│   └── figures/
├── docker/
│   ├── dockerfile
│   └── entrypoint.sh
├── config/config/
│   ├── launch_setup.yaml
│   ├── remappings.yaml
│   ├── package_config/
│   └── vehicle_config/
├── src/
│   ├── tod_launch/
│   └── ...
└── .dev_artifacts/
    ├── build/
    ├── install/
    └── log/
```

项目按主题分为多个模块：

| 模块 | 说明 |
|---|---|
| `tod_common` | 公共工具、参数处理等 |
| `tod_msgs` | 自定义消息和服务定义 |
| `tod_state_machine` | operator / vehicle 状态机与安全状态切换 |
| `tod_direct_control` | 直接遥控、命令生成与转发 |
| `tod_trajectory_guidance` | 轨迹引导、轨迹跟踪和 pure pursuit 控制 |
| `tod_network` | 网络传输、数据封装、话题转发 |
| `tod_operator_interface` | 操作员界面、输入设备、视频管理 |
| `tod_vehicle_interface` | 车辆接口与车辆仿真 |
| `tod_perception` | 感知数据处理 |
| `tod_safety` | 安全链路 |
| `tod_logging` | 日志记录 |
| `tod_monitoring` | 状态与网络监控 |
| `tod_launch` | 统一 launch 与参数配置 |

核心配置文件：

- `config/config/launch_setup.yaml`
  - 选择车辆 ID、仿真/实车模式。
  - 配置 operator、vehicle、both 三类启动包。
- `config/config/remappings.yaml`
  - 定义各模块之间的 topic remap 关系。
- `config/config/package_config/`
  - 各 ROS 2 节点的参数文件。
- `config/config/vehicle_config/`
  - 每辆车或仿真平台的参数，例如 `edgar`、`edgarautoware`、`rc-car`。

## 一周教学课程

以下课程建议每天安排 2 到 4 小时。前两天偏环境与运行，后三天偏源码与数据流。

### 第 1 天：项目概览与 Docker 环境运行

**目标**

- 理解项目定位和系统架构。
- 明确 Docker 隔离原则，确保不在 DGX 宿主机安装 ROS。
- 跑通最小仿真。

**阅读**

- `README.md`
- `doc/figures/tod_architecture.png`
- `doc/getting_started.md`
- `doc/docker_workflow.md`
- 本文档的项目结构和 Docker 运行方式章节

**实验**

1. 查看当前镜像：

   ```bash
   docker image ls tod_operator_base
   ```

2. 启动容器：

   ```bash
   docker run --rm -it \
     --user 1000:1000 \
     --shm-size=1g \
     --network none \
     -v "$PWD/src:/home/tum/wsp/src" \
     -v "$PWD/config:/home/tum/wsp/config" \
     -v "$PWD/.dev_artifacts/build:/home/tum/wsp/build" \
     -v "$PWD/.dev_artifacts/install:/home/tum/wsp/install" \
     -v "$PWD/.dev_artifacts/log:/home/tum/wsp/log" \
     tod_operator_base:latest bash
   ```

3. 启动最小仿真：

   ```bash
   source /opt/ros/humble/setup.bash
   source install/setup.bash
   ros2 launch tod_launch tod_both.launch.py
   ```

4. 新开一个容器终端，观察节点与话题：

   ```bash
   ros2 node list
   ros2 topic list
   ```

**作业**

- 画出 operator 端、vehicle 端和共享模块的大致关系。
- 标出 `launch_setup.yaml` 中属于 operator、vehicle、both 的包。
- 回答：为什么 `only_sim` 模式不需要实车接口？

### 第 2 天：启动系统与参数配置

**目标**

- 理解 `tod_launch` 如何把 YAML 配置转换为 ROS 2 launch。
- 掌握 operator、vehicle、both 三种启动方式。
- 学会用 ROS 2 CLI 观察系统。

**阅读**

- `src/tod_launch/README.md`
- `src/tod_launch/launch/tod_operator.launch.py`
- `src/tod_launch/launch/tod_vehicle.launch.py`
- `src/tod_launch/launch/tod_both.launch.py`
- `src/tod_launch/utils/parse_launch_setup.py`
- `src/tod_launch/utils/parse_remappings.py`
- `src/tod_launch/utils/create_launch_description.py`
- `config/config/launch_setup.yaml`
- `config/config/remappings.yaml`

**重点概念**

- `vehicleID`
  - 决定加载哪套车辆参数。
  - `edgar` 是默认车辆。
- `mode`
  - `only_sim` 加载 `tod_vehicle_sim`。
  - `vehicle` 加载对应实车接口，例如 `tod_edgar_interface`。
- `packages_to_launch`
  - 决定哪些包启动。
  - operator、vehicle、both 分别表示启动位置。

**实验**

1. 启动完整仿真：

   ```bash
   ros2 launch tod_launch tod_both.launch.py
   ```

2. 单独启动 operator：

   ```bash
   ros2 launch tod_launch tod_operator.launch.py
   ```

3. 单独启动 vehicle：

   ```bash
   ros2 launch tod_launch tod_vehicle.launch.py
   ```

4. 观察状态话题：

   ```bash
   ros2 topic echo /vehicle/statemachine/output/vehicle_status
   ```

5. 观察网络监控话题：

   ```bash
   ros2 topic echo /vehicle/network/data/to_operator/network_metrics
   ```

6. 查看节点信息：

   ```bash
   ros2 node info /vehicle/trajectory_guidance/control/PurePursuitSimulator
   ```

**作业**

- 写出 `only_sim` 模式下加载的 vehicle interface 包名。
- 总结 `remappings.yaml` 在系统中的作用。
- 找出与 `/vehicle/statemachine/output/vehicle_status` 相连的节点。

### 第 3 天：消息、状态机与安全链路

**目标**

- 理解项目自定义消息。
- 理解 operator 和 vehicle 状态机。
- 掌握安全状态切换和远程驾驶的启动流程。

**阅读**

- `src/tod_msgs/`
- `src/tod_state_machine/tod_state_machine/`
- `src/tod_safety/tod_safety_gate/`
- `config/config/package_config/tod_safety_gate/`

**重点概念**

- `tod_msgs`
  - 定义系统内部传递的状态、控制、轨迹等数据类型。
- `tod_state_machine`
  - operator 端状态机：管理操作员请求和界面状态。
  - vehicle 端状态机：管理车辆实际状态、允许控制和紧急停止。
- `tod_safety`
  - safety gate 负责判断控制命令是否可执行。

**实验**

1. 查看消息定义：

   ```bash
   find src/tod_msgs -type f | sort
   ```

2. 查看状态话题：

   ```bash
   ros2 topic echo /operator/statemachine/output/operator_status
   ros2 topic echo /vehicle/statemachine/output/vehicle_status
   ```

3. 查看安全相关话题：

   ```bash
   ros2 topic list | grep safety
   ```

4. 阅读状态机源码，标出以下状态转换：
   - 空闲
   - 连接
   - 控制中
   - 紧急停止
   - 恢复

**作业**

- 画出一个远程驾驶会话的状态转换图。
- 标出哪些状态由 operator 发起，哪些状态由 vehicle 确认。
- 说明 safety gate 在命令执行前做了哪些检查。

### 第 4 天：直接遥控、轨迹引导与车辆仿真

**目标**

- 理解两种主要远程驾驶概念：直接遥控和轨迹引导。
- 理解控制命令从 operator 到 vehicle 的路径。
- 理解 `tod_vehicle_sim` 的仿真行为。

**阅读**

- `src/tod_direct_control/tod_command_creation/`
- `src/tod_direct_control/tod_command_forwarder/`
- `src/tod_trajectory_guidance/tod_pure_pursuit/`
- `src/tod_trajectory_guidance/tod_trajectory_guidance/`
- `src/tod_vehicle_interface/`
- `config/config/package_config/tod_command_creation/`
- `config/config/package_config/tod_pure_pursuit/`
- `config/config/package_config/tod_trajectory_guidance/`
- `config/config/vehicle_config/edgar/`

**重点概念**

- Direct Control
  - 操作员直接输入速度、转向等控制量。
  - 命令经过 command creation、network、forwarder、safety gate 后进入车辆仿真或实车接口。
- Trajectory Guidance
  - 操作员在界面规划轨迹。
  - vehicle 端执行轨迹跟踪。
  - pure pursuit 控制器根据轨迹生成控制命令。
- `tod_vehicle_sim`
  - `only_sim` 模式下的车辆动力学仿真。
  - 用于无实车环境开发与测试。

**实验**

1. 观察控制话题：

   ```bash
   ros2 topic list | grep control
   ```

2. 观察轨迹话题：

   ```bash
   ros2 topic list | grep trajectory
   ```

3. 查看纯跟踪控制器参数：

   ```bash
   ros2 param list /vehicle/trajectory_guidance/control/PurePursuitSimulator
   ```

4. 查看轨迹输出：

   ```bash
   ros2 topic echo /vehicle/trajectory_guidance/output/validation_trajectory
   ```

5. 如果有 GUI，分别测试 Direct Control 和 Trajectory Guidance。

**作业**

- 写出 Direct Control 的数据流路径。
- 写出 Trajectory Guidance 的数据流路径。
- 说明 pure pursuit 的 lookahead 参数如何影响轨迹跟踪。

### 第 5 天：网络传输、视频与操作员界面

**目标**

- 理解 operator 与 vehicle 之间的数据传输设计。
- 理解视频流和点云流的传输方式。
- 掌握 operator 界面组成和输入设备关系。

**阅读**

- `src/tod_network/tod_communication_interface/`
- `src/tod_network/tod_data_interface/`
- `src/tod_network/tod_lidar/`
- `src/tod_network/tod_network/`
- `src/tod_network/tod_rtsp/`
- `src/tod_operator_interface/tod_input_devices/`
- `src/tod_operator_interface/tod_visual/`
- `src/tod_monitoring/tod_network_monitoring/`
- `src/tod_monitoring/tod_topic_monitoring/`
- `config/config/package_config/tod_communication_interface/`
- `config/config/package_config/tod_data_interface/`
- `config/config/package_config/tod_network_monitoring/`
- `config/config/package_config/tod_rtsp/`
- `config/config/package_config/tod_visual/`

**重点概念**

- 网络传输
  - vehicle 端发送状态、传感器和视频数据。
  - operator 端发送控制命令和配置请求。
  - `generic_tod_sender` / `generic_tod_receiver` 负责序列化与传输。
- 视频链路
  - vehicle 端 RTSP server。
  - operator 端 RTSP client。
  - `tod_visual` 和 `video_manager` 负责显示与管理。
- 网络监控
  - network monitor 输出延迟、带宽等信息。
  - packet logger 可用于抓包和链路分析。

**实验**

1. 查看网络相关话题：

   ```bash
   ros2 topic list | grep network
   ```

2. 观察网络指标：

   ```bash
   ros2 topic echo /vehicle/network/data/to_operator/network_metrics
   ```

3. 查看视频话题：

   ```bash
   ros2 topic list | grep video
   ```

4. 查看点云话题：

   ```bash
   ros2 topic list | grep pointcloud
   ```

5. 如果有 GUI，启动完整界面并观察视频窗口、可视化窗口和输入设备窗口。

**作业**

- 画出视频从 vehicle 到 operator 的传输路径。
- 画出控制命令从 operator 到 vehicle 的传输路径。
- 说明网络监控数据如何进入状态机或安全链路。

## 进阶方向

完成一周课程后，可按以下方向继续学习：

1. 实车接口与车辆平台
   - 阅读 `tod_edgar_interface`。
   - 研究 `vehicle_config` 中不同车辆平台的参数差异。
   - 理解实车接口与仿真接口的替换关系。
2. Autoware / Tier4 集成
   - 研究 `tod_edgarautoware_interface`。
   - 补齐 Autoware 相关消息依赖。
   - 理解自动驾驶系统与远程驾驶系统之间的状态同步。
3. 性能与网络优化
   - 分析视频编码参数。
   - 研究点云压缩与网络监控指标。
   - 使用 rosbag 记录并回放数据。
4. 安全与可靠性
   - 深入阅读 state machine 和 safety gate。
   - 设计故障注入实验。
   - 分析网络中断、延迟和丢包时的系统行为。

## 常用命令

```bash
# 查看镜像
docker image ls tod_operator_base

# 查看容器
docker ps

# 进入运行中的容器
docker exec -it <container_id> bash

# 查看 ROS 2 节点
ros2 node list

# 查看 ROS 2 话题
ros2 topic list

# 查看某个话题数据
ros2 topic echo /vehicle/statemachine/output/vehicle_status

# 查看节点详细信息
ros2 node info /vehicle/trajectory_guidance/control/PurePursuitSimulator

# 查看参数
ros2 param list /vehicle/trajectory_guidance/control/PurePursuitSimulator

# 构建全部可构建包
colcon build \
  --packages-ignore tod_dummy tod_edgar_interface tod_edgarautoware_interface

# 构建指定包
colcon build --packages-select tod_launch tod_state_machine
```

## 排障

### GUI 无法启动

现象：

```text
Failed to create GLFW window
This application failed to start because no Qt platform plugin could be initialized
```

原因：

- 宿主机没有 X11 环境。
- 容器没有挂载 `/tmp/.X11-unix`。
- `DISPLAY` 或 `XAUTHORITY` 配置不正确。

处理：

- 使用带 GUI 的 `docker run` 命令。
- 或在 `launch_setup.yaml` 中关闭 `tod_visual` 和 `tod_input_devices`，只验证核心链路。

### 虚拟摇杆 QML 报错

现象：

```text
file:///home/tum/wsp/build/tod_input_devices/src/virtual_joystick.qml: No such file or directory
```

原因：

- 程序从 build 目录读取 QML 文件。
- 实际安装位置为 `install/tod_input_devices/lib/tod_input_devices/virtual_joystick.qml`。

影响：

- 虚拟摇杆 UI 无法正常显示。
- 不影响核心仿真、状态机、轨迹跟踪和网络链路。

### rosdep 找不到 Autoware 消息包

现象：

```text
Cannot locate rosdep definition for autoware_auto_vehicle_msgs
```

处理：

- 最小仿真模式无需这些包。
- 构建时使用 `--skip-keys`。
- 或后续单独安装 Autoware / Tier4 消息包。

### 容器内 sudo 提示主机名解析失败

现象：

```text
sudo: unable to resolve host <container_id>: Temporary failure in name resolution
```

原因：

- 运行时使用了 `--network none`，entrypoint 中 `sudo` 尝试解析容器主机名。

影响：

- 不影响核心仿真运行。
- 只是无 DNS 时 sudo 的提示信息。

### 服务转发器提示 Network is unreachable

现象：

```text
Connecting to ip: 10.0.0.10 port: 60300
connect: Network is unreachable
```

原因：

- 本地 `tod_both.launch.py` 一体仿真中，operator 和 vehicle 都在同一个容器内。
- 车辆 IP 被设置为 `10.0.0.10` 等非本机地址。
- 如果容器使用 `--network none`，非 loopback 地址必然不可达。

处理：

- 在 Manager 窗口中将 operator 和 vehicle IP 都设置为 `127.0.0.1`。
- 当前源码已将 Manager 的车辆 IP 默认值修复为 `127.0.0.1`。
- 修复后，`60100`、`60300`、`60400` 及对应的 `60101`、`60301`、`60401` 服务连接应正常建立。

验证方法：

```bash
# 在第二个容器终端中执行
source /opt/ros/humble/setup.bash
source install/setup.bash

ros2 service call \
  /operator/network/config/to_vehicle/set_monitoring_status \
  tod_network_monitoring_msgs/srv/NetworkMonitorService \
  "{vehicle_ip_address: 127.0.0.1, set_monitor_mode: false}"
```

如果返回类似以下内容，说明 operator 到 vehicle 的服务转发链路已通：

```text
tod_network_monitoring_msgs/srv.NetworkMonitorService_Response(is_active=False)
```

### PacketLogger 提示 capabilities 设置失败

现象：

```text
Could not set capabilities to PacketLogger! Try building first and then source again
```

处理：

- 通常出现在 install 产物尚未生成或路径未就绪时。
- 核心仿真运行可忽略。
- 如需抓包功能，需先完成构建并确认 PacketLogger 二进制存在。
