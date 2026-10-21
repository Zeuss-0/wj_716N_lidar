# wj_716N_lidar

万集（Wanji）716N 二维激光雷达的 ROS 1（catkin）驱动。节点通过 TCP 连接雷达，把扫描发布为 `sensor_msgs/LaserScan`。

ROS 1 (catkin) driver for the Wanji 716N 2D lidar. The node connects over TCP and publishes scans as `sensor_msgs/LaserScan`.

本仓库没有厂商手册。下文凡是「启动文件中的取值」，只说明本仓库代码和 launch 文件里实际写了什么，不代表万集出厂默认值。代码来源和再分发限制见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。

This repository does not include a vendor manual. "Launch-file settings" below mean only what this tree contains, not Wanji factory defaults. Origin and redistribution limits are in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## 许可 / License

`package.xml` 声明本包为 **Apache 2.0**。根目录 [`LICENSE`](LICENSE) 是与该声明对应的 Apache License 2.0 全文。

这**不能**当成万集原始驱动已经按 Apache-2.0 授权：

- `src/`、`cfg/`、`launch/` 来自万集 716N / 万集 mini 雷达的厂商 demo 或 SDK，经 ROS 社区再打包。同一代码族的另一份发布在 <https://github.com/MagicPrince666/ros2_wj_716N_lidar>。
- 这些源文件**没有**版权声明或许可证头。
- 原始厂商代码能否再分发并不清楚。需要再分发时请向万集科技确认。
- 根目录 `LICENSE` **没有**把上述厂商文件重新许可为 Apache-2.0。详情以 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) 为准。

`package.xml` declares **Apache 2.0**, and the root [`LICENSE`](LICENSE) file is the Apache License 2.0 text for that declaration.

That does **not** mean the original Wanji driver is Apache-2.0:

- `src/`, `cfg/`, and `launch/` come from Wanji 716N / Wanji mini-lidar vendor demo or SDK code, repackaged in the ROS community. Another publication of the same family is <https://github.com/MagicPrince666/ros2_wj_716N_lidar>.
- Those sources have **no** copyright or license header.
- Redistribution rights for the vendor code are unclear. Confirm with WanJi Technology before redistributing it.
- The root `LICENSE` does **not** re-license those vendor files as Apache-2.0. [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) is the statement of that limit.

## 功能 / What the node does

可执行文件名是 `wj_716N_lidar`。

1. 从节点私有参数读取 `hostname` 和 `port`，用 TCP 连接雷达。连不上则每 5 秒重试一次。
2. 解析扫描包。线上距离是毫米，发布前除以 1000，单位为米。超出 `range_min` / `range_max`，或者读数为 0 的点，记为 NaN。
3. 在全局话题 `scan` 上发布 `sensor_msgs/LaserScan`。launch 文件可以用 remap 改名。
4. 用 dynamic_reconfigure 更新角度、量程、扫描模式和 `frame_id`。
5. 设备上报的频率字节如果和参数 `frequency_scan` 不一致，这一帧会被丢掉，终端打印 `The scan frequency does not match the one you setted!`。
6. 主循环每 2 秒检查一次。这一拍里如果没有收到有效扫描包，节点会断开，并在后台线程里重连。

驱动按整幅窗口索引原始点：±135°（±2.35619449 rad），1081 点，角增量 `0.017453 / 4` rad（0.25°）。`min_ang` / `max_ang` 只裁剪发布出去的那一段。

Executable name: `wj_716N_lidar`.

1. Reads the private parameters `hostname` and `port` and opens a TCP connection. A failed connect is retried every 5 seconds.
2. Parses scan packets. On-wire distances are millimeters; the driver divides by 1000 and publishes meters. Readings outside `range_min` / `range_max`, or equal to 0, are stored as NaN.
3. Publishes `sensor_msgs/LaserScan` on the global topic `scan`. Launch files may remap that name.
4. Applies angle, range, scan mode, and `frame_id` through dynamic_reconfigure.
5. Drops a frame when the frequency byte from the device does not match `frequency_scan`, and prints `The scan frequency does not match the one you setted!`.
6. The main loop checks a heartbeat every 2 seconds. If that interval had no valid scan packet, the node disconnects and reconnects on a background thread.

Raw points are indexed over ±135° (±2.35619449 rad), 1081 returns, with increment `0.017453 / 4` rad (0.25°). `min_ang` / `max_ang` only crop what is published.

## 硬件说明 / Hardware

本仓库**没有** 716N 的说明书、供电或接线说明。这些请以设备铭牌和万集资料为准。

从驱动和启动文件能确定的内容如下。

| 项目 | 本仓库中的情况 |
| --- | --- |
| 链路 | TCP。单雷达示例 `192.168.1.20`，端口 `2110` |
| 第二台 | `double_lidar.launch` 使用 `192.168.1.21`，端口同样是 `2110` |
| 角分辨率 | 代码按 0.25° |
| 扫描模式 | `frequency_scan=1`：0.25° / 15 Hz；`=2`：0.25° / 25 Hz。设备实际模式必须和参数一致，否则扫描会被丢弃 |
| 全幅视场 | 按 ±135° 索引，1081 点 |
| 量程 | 发布单位是米。示例把 `range_max` 设为 30 m（单雷达）或 25 m（双雷达）。这是参数上限，不是说明书额定测距 |

本仓库不能证明出厂地址就是 `192.168.1.20:2110`。连不上时修改 launch 里的 `hostname`，并让电脑网卡和雷达处在同一网段。

This repository does **not** include a 716N manual, power rating, or wiring diagram. Use the nameplate and Wanji's own documentation for those.

| Item | In this repository |
| --- | --- |
| Link | TCP. Single-lidar example `192.168.1.20`, port `2110` |
| Second lidar | `double_lidar.launch` uses `192.168.1.21`, also port `2110` |
| Angular step | Coded as 0.25° |
| Scan mode | `frequency_scan=1` is 0.25° / 15 Hz; `2` is 0.25° / 25 Hz. The device mode must match or frames are dropped |
| Full window | Indexed as ±135°, 1081 points |
| Range | Published in meters. Samples set `range_max` to 30 m (one lidar) or 25 m (two). That is a parameter cap, not a datasheet rating |

Nothing here proves that `192.168.1.20:2110` is the factory address. If the sensor is not found, edit `hostname` and put the computer on the same subnet.

## 依赖 / Dependencies

ROS 1 + catkin。`CMakeLists.txt` 的写法对应 Kinetic 及之后的 ROS 1。`cfg/wj_716N_lidar.cfg` 的 shebang 是 `python`。Noetic 上如果没有 `/usr/bin/python`，配置生成会失败，需要提供 `python` 命令（例如安装 `python-is-python3`）。

`package.xml` 中的编译和运行依赖：

- `roscpp`
- `sensor_msgs`
- `visualization_msgs`（`wj_716N_lidar_protocol.h` 包含了 `visualization_msgs/Marker.h`）
- `dynamic_reconfigure`
- Boost（Asio、thread、bind），通常随 `roscpp` 一起安装

只有 `launch/double_lidar.launch` 还需要单独的功能包 [`ira_laser_tools`](http://wiki.ros.org/ira_laser_tools)。单雷达启动不需要它。

```bash
sudo apt install ros-noetic-roscpp ros-noetic-sensor-msgs \
  ros-noetic-visualization-msgs ros-noetic-dynamic-reconfigure
# 需要合并双雷达时再安装：
# sudo apt install ros-noetic-ira-laser-tools
```

Melodic 把包名里的 `noetic` 换成 `melodic`。

ROS 1 with catkin, in the style `CMakeLists.txt` marks as Kinetic and newer. The cfg script's shebang is `python`, so Noetic needs a `python` executable on `PATH` (for example `python-is-python3`) or dynamic-reconfigure generation fails.

Build and run dependencies from `package.xml`:

- `roscpp`
- `sensor_msgs`
- `visualization_msgs` (`wj_716N_lidar_protocol.h` includes `visualization_msgs/Marker.h`)
- `dynamic_reconfigure`
- Boost (Asio, thread, bind), usually pulled in with `roscpp`

Only `launch/double_lidar.launch` needs the separate package [`ira_laser_tools`](http://wiki.ros.org/ira_laser_tools).

## 编译 / Build

把本包放进 catkin 工作空间的 `src/`，在工作空间根目录编译。

```bash
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/src
git clone https://github.com/Zeuss-0/wj_716N_lidar.git
cd ~/catkin_ws
source /opt/ros/noetic/setup.bash
catkin_make
source devel/setup.bash
```

Place this package in a catkin workspace `src/` directory and build from the workspace root, as above. Source the distro you actually have installed (`melodic` or `noetic`) before `catkin_make`.

## 启动 / Launch

先 `source devel/setup.bash`，再按雷达实际地址修改 `hostname`。

### 单雷达 / One lidar

```bash
roslaunch wj_716N_lidar wj_716N_lidar_01.launch
```

`launch/wj_716N_lidar_01.launch` 中的取值：

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `hostname` | `192.168.1.20` | 雷达 IP |
| `port` | `2110` | 字符串，程序里用 `atoi` 转成端口号 |
| `frequency_scan` | `1` | 0.25° / 15 Hz |
| `min_ang` | `-2.00712864` | **−115°**。驱动全幅默认是 −135°，这个 launch 裁得更窄 |
| `max_ang` | `2.00712864` | **+115°** |
| `frame_id` | `laser` | 写入扫描消息的 frame，本包不广播 TF |
| `range_min` | `0.05` | 米 |
| `range_max` | `30.0` | 米 |

节点名是 `wj_716N_lidar_01`，话题是 `/scan`。

RViz 里把 Fixed Frame 设为 `laser`，添加 LaserScan，Topic 选 `/scan`。本包不发布 TF。Fixed Frame 需要等于 `frame_id`，或者由别的节点提供两者之间的变换。

Node name `wj_716N_lidar_01`, topic `/scan`. This package does not broadcast TF, so the RViz fixed frame must be `laser` or another node must publish the transform.

### 双雷达 / Two lidars

```bash
roslaunch wj_716N_lidar double_lidar.launch
```

| 节点 | IP | `frame_id` | 话题 | 角度 | `range_max` |
| --- | --- | --- | --- | --- | --- |
| `front_laser` | `192.168.1.20` | `front_laser` | `/front_scan` | ±110°（±1.919862177 rad） | 25 m |
| `rear_laser` | `192.168.1.21` | `rear_laser` | `/rear_scan` | ±110°（±1.919862177 rad） | 25 m |

两台端口都是 `2110`。这个文件没有设置 `frequency_scan`，因此使用 cfg 默认值 `1`（15 Hz）。`range_min` 仍是 `0.05` m。

文件末尾没有额外参数，直接包含：

```xml
<include file="$(find ira_laser_tools)/launch/laserscan_multi_merger.launch" />
```

合并节点的输入话题、输出话题和目标坐标系来自 `ira_laser_tools` 自己的默认 launch，**不会**自动对准 `/front_scan` 和 `/rear_scan`。使用前请打开那个 launch，把激光话题改成这两路扫描，并补上它要求的 TF。

Both ports are `2110`. `frequency_scan` is unset, so the cfg default `1` (15 Hz) is used. `range_min` stays `0.05` m.

The merger include passes no arguments. Its topics and destination frame are whatever `ira_laser_tools` defaults to, which are not `/front_scan` and `/rear_scan`. Align that launch and supply the TF it expects before relying on a merged scan.

## 参数 / Parameters

`hostname` 和 `port` 只从私有参数读取（`~hostname`、`~port`），源码里没有默认值。参数为空时连接失败，然后每 5 秒重试。

其余项是 dynamic_reconfigure 参数，也在私有命名空间。`cfg/wj_716N_lidar.cfg` 的默认值：

| 参数 | 类型 | cfg 默认 | 范围 | 含义 |
| --- | --- | --- | --- | --- |
| `min_ang` | double | `-3.14` | ±3.14 rad | 发布扫描的起始角 |
| `max_ang` | double | `3.14` | ±3.14 rad | 发布扫描的终止角 |
| `range_min` | double | `0` | 0–30 m | 最小量程 |
| `range_max` | double | `30` | 0–30 m | 最大量程 |
| `frequency_scan` | int | `1` | 1 或 2 | `1` = 0.25° / 15 Hz，`2` = 0.25° / 25 Hz |
| `frame_id` | string | `laser` | | `LaserScan.header.frame_id` |

对象刚构造、dynamic_reconfigure 尚未回调时，`frame_id` 先是 `wj_716N_lidar_frame`，角度先是 ±135°。服务器启动时会套用参数，因此正常 `roslaunch` 以 launch 文件和 cfg 默认值为准。

`hostname` and `port` are private parameters with no default in the source. An empty value fails to connect, then retries every 5 seconds.

The remaining parameters are dynamic_reconfigure values on the private namespace. The table is `cfg/wj_716N_lidar.cfg`.

Until the reconfigure callback runs, the constructor uses `frame_id` `wj_716N_lidar_frame` and angles ±135°. The server applies parameters at startup, so a normal `roslaunch` follows the launch file and the cfg defaults.

## 话题 / Topics

| 话题 | 类型 | 说明 |
| --- | --- | --- |
| `scan` | `sensor_msgs/LaserScan` | 全局名。单雷达 launch 下是 `/scan` |
| `front_scan`、`rear_scan` | 同上 | 只在双雷达 launch 里由 remap 得到 |

没有订阅，也没有服务。dynamic_reconfigure 使用该节点私有命名空间下的标准接口。

There are no subscriptions and no services. dynamic_reconfigure uses the node's private namespace.

## 维护者 / Maintainer

`package.xml` 里原来的 `WJ@todo.todo` 是占位符。仓库中没有万集官方维护者邮箱，因此维护者写的是本仓库所有者 **Jiuxu Yang**（GitHub [Zeuss-0](https://github.com/Zeuss-0)）。这只标识谁维护这份 ROS 包，不表示万集驱动源码的著作权归属。

The old `WJ@todo.todo` entry was a placeholder. No vendor maintainer address is in this tree, so the manifest lists **Jiuxu Yang** (GitHub [Zeuss-0](https://github.com/Zeuss-0)). That names who maintains this ROS package. It does not assign copyright in the Wanji driver sources.
