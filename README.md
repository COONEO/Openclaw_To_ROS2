# OpenClaw x ROS2 机械夹爪复现

![](https://raw.githubusercontent.com/COONEO/Openclaw_To_ROS2/refs/heads/main/1.jpg)

本文用于从 0 到 1 复现：通过 OpenClaw 在 ROS2 中控制机械夹爪开合和角度。

## 1. 环境搭建与下载

- Ubuntu 22.04
- ROS2 Humble
- OpenClaw（安装说明：https://openclaws.io/zh/）
- RosClaw 仓库：https://github.com/PlaiPin/rosclaw
- 功能包仓库：https://github.com/COONEO/Openclaw_To_ROS2

安装基础依赖：

```bash
sudo apt update
sudo apt install -y git curl build-essential python3-colcon-common-extensions ros-humble-rosbridge-server
npm install -g openclaw@latest
openclaw onboard --install-daemon
npm install -g pnpm@9.15.4
```

下载代码：

```bash
mkdir -p <your_ws>
cd <your_ws>
git clone https://github.com/COONEO/Openclaw_To_ROS2 serial_servo
git clone https://github.com/PlaiPin/rosclaw.git rosclaw
```

## 2. 安装 RosClaw 插件并应用补丁

```bash
cd <your_ws>/rosclaw
pnpm install
openclaw plugins install -l <your_ws>/rosclaw/extensions/openclaw-plugin
openclaw plugins enable rosclaw
git apply <your_ws>/serial_servo/rosclaw_local.patch
pnpm install
openclaw config set plugins.entries.rosclaw.config '{"transport":{"mode":"rosbridge"},"rosbridge":{"url":"ws://127.0.0.1:9090"},"robot":{"name":"serial-gripper","namespace":""}}'
openclaw gateway restart
```

## 3. 启动通信与控制链路

终端 A：编译并启动串口桥节点

<p align="center">
  <img src="https://raw.githubusercontent.com/COONEO/Openclaw_To_ROS2/refs/heads/main/Screenshot%20from%202026-03-05%2017-34-03.png" width="800">
</p>

```bash
cd <your_ws>/serial_servo
source /opt/ros/humble/setup.bash
colcon build
source install/setup.bash
ros2 launch serial_servo_bridge serial_servo_bridge.launch.py
```

终端 B：启动 rosbridge

```bash
source /opt/ros/humble/setup.bash
source <your_ws>/serial_servo/install/setup.bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
```

终端 C：加载 skill 并打开 OpenClaw 控制台

```bash
mkdir -p ~/.openclaw/skills/gripper-control
cp <your_ws>/serial_servo/openclaw_skills/gripper-control/SKILL.md ~/.openclaw/skills/gripper-control/SKILL.md
openclaw gateway restart
openclaw dashboard
```

## 4. OpenClaw 控制机械夹爪

在 OpenClaw Dashboard 会话中发送以下指令即可控制：

- `打开夹爪`
- `闭合夹爪`
- `把夹爪调到 -20 度`
- `当前夹爪角度是多少`

也可以直接发送工具指令：

`用 ros2_publish 向 /servo/command 发布 serial_servo_bridge/msg/ServoCommand，消息为 {mode: 0, servo_id: 1, value: -30.0}`

## 5. 关键文件说明

- [openclaw_skills/gripper-control/SKILL.md](./openclaw_skills/gripper-control/SKILL.md)：夹爪控制 skill
- [rosclaw_local.patch](./rosclaw_local.patch)：RosClaw 本地补丁
- [serial_servo_bridge/README.md](./serial_servo_bridge/README.md)：串口桥节点说明
