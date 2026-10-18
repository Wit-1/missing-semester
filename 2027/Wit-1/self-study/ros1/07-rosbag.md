# 07：rosbag 录制与回放

## 学了什么

rosbag 可以将 ROS 话题消息保存到 `.bag` 文件中，之后按录制时的时间间隔重新发布。文件保存话题名、消息类型、内容和记录时间。

视觉项目可以录下一段相机图像，再用同一段数据反复验证算法。本次先用海龟的位置和速度指令练习。

| 操作 | rosbag 的作用 |
| --- | --- |
| `rosbag record` | 订阅话题，将收到的消息写入文件 |
| `rosbag info` | 查看文件中的话题、类型、消息数量和时长 |
| `rosbag play` | 读取文件，将记录的消息重新发布 |

## 做过的实验：录制位置和速度指令

所有命令在容器开发终端运行。保持 `lesson.launch` 和海龟窗口运行，先结束上一轮尚未停止的运动控制程序。

在一个开发终端开始录制：

```bash
mkdir -p ~/catkin_ws/records
cd ~/catkin_ws/records
rosbag record -O turtle_demo.bag /turtle1/pose /turtle1/cmd_vel
```

`-O` 指定输出文件名。看到 `Recording to ...` 后，在另一个开发终端复位并让海龟画圆：

```bash
rosservice call /reset
rosrun turtle_control turtle_motion _linear:=1 _angular:=1 _duration:=6.3
```

等运动结束后，在录制终端按 Ctrl+C，等待文件保存完成，再查看：

```bash
rosbag info turtle_demo.bag
```

文件在容器中的路径为：

```text
/home/rosdev/catkin_ws/records/turtle_demo.bag
```

对应宿主机的 `DynamicX/catkin_ws/records/turtle_demo.bag`。录制数据保存在本地开发目录，学习笔记中保留命令和结果。

## 实际录制结果

| 项目 | 本次结果 |
| --- | --- |
| 文件格式版本 | 2.0 |
| 时长 | 54.2 秒 |
| 大小 | 276.9 KB |
| 消息总数 | 3504 |
| 压缩 | 无 |

`version: 2.0` 是 bag 文件格式版本，本次使用的开发环境仍是 ROS 1 Noetic。

| 话题 | 消息数量 | 消息类型 |
| --- | --- | --- |
| `/turtle1/cmd_vel` | 130 | `geometry_msgs/Twist` |
| `/turtle1/pose` | 3374 | `turtlesim/Pose` |

54.2 秒是整个录制时长，包括运动前后的静止时间。位置消息在海龟停着时也持续发布；速度指令主要由控制程序在运动期间和结束停车时发送，所以两种消息的数量不同。

## 做过的实验：用原来的 listener 接收回放数据

保持 `lesson.launch` 运行，使当前 ROS master 可用。在另一个开发终端启动专门接收回放位置的实例：

```bash
rosrun turtle_control turtle_pose_listener \
  __name:=replay_listener \
  turtle1/pose:=/replay/turtle1/pose
```

`__name:=replay_listener` 设置这次运行的节点名；后面的重映射让它订阅 `/replay/turtle1/pose`。回放开始前，它只显示正在监听并等待消息。

在另一个开发终端回放：

```bash
rosbag play ~/catkin_ws/records/turtle_demo.bag \
  /turtle1/pose:=/replay/turtle1/pose \
  --topics /turtle1/pose
```

`--topics /turtle1/pose` 选择文件中的原始位置话题；重映射将发布目标改为 `/replay/turtle1/pose`。本次已确认原来的 listener 能重新输出历史位置。

```text
turtle_demo.bag
    → rosbag 回放节点读取并发布
    → /replay/turtle1/pose
    → replay_listener 接收并打印
```

## 观察与解释

listener 的 C++ 代码没有变化。它仍然按 `turtlesim/Pose` 类型处理消息，只是通过重映射更换了消息来源。回放节点负责发布，listener 负责接收。

真实海龟的位置继续使用 `/turtle1/pose`，回放数据使用 `/replay/turtle1/pose`，两种来源因此可以分别观察。

本次只回放位置数据，海龟窗口保持当前状态，listener 输出记录中的历史坐标。要驱动海龟运动，需要向它订阅的速度话题发送指令；位置消息本身不会控制运动。

默认播放时长约 54 秒，静止时段也按记录回放，因此有些阶段会连续打印相同坐标。回放结束后，listener 仍保持运行，继续等待新消息。
