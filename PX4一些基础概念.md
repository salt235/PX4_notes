# PX4一些基础概念

## 1. PX4架构

### 以飞控为中心的简单系统

![image-20260701140303622](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260701140303622.png)

- PX4 和外界所有通信都必须经过 MAVLink（或者封装后的 MAVSDK/ROS）

- RC 信号优先级通常比 Offboard 高（安全机制）

- PX4 内部所有模块都通过 uORB 交换数据（uORB可以理解为一个轻量的飞控内部的 ROS topic 系统））

- Commander是应用层中的“总控状态机”，负责：飞行模式管理（Manual / Offboard / Auto），安全检查（解锁条件），起飞降落逻辑。

### 飞控加机载电脑的系统

  ![image-20260701141139316](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260701141139316.png)

核心区别：**单独飞控架构里，PX4 飞控既是飞行控制核心，也是主要计算中心；飞控加伴随计算机架构里，PX4 飞控只负责底层飞行稳定，复杂任务被转移到 Linux 电脑上。**

## 2. PX4涉及的一些通信方式

### uORB（PX4内部模块之间的通信）

这是 PX4 最核心的内部通信机制，它和 ROS 的 topic 很像，但更轻量，运行在飞控内部。

### MAVLink（PX4 和外部世界通信）

MAVLink 是 PX4 最常见的外部通信协议。

### UDP / TCP（仿真和网络通信）

UDP/TCP 主要用于仿真、网络连接、机载电脑和地面站之间的 MAVLink 传输。

### PWM（电机输出控制）

PWM = Pulse Width Modulation（脉宽调制），用“高电平持续时间”表示控制量，主要用来控制电机，舵机。

### CAN（工业级总线，PX4未来趋势）

CAN = Controller Area Network，一种“多设备共享的高速抗干扰总线”，一条总线挂很多设备（像“局域网”），可以连一些高级传感器，比如GPS，测距等等。

### I2C（低速传感器总线）

I2C = Inter-Integrated Circuit，两根线连接多个低速设备的通信协议，适用一些典型传感器，比如气压，磁力，距离等等。

### SPI（高速传感器）

SPI = Serial Peripheral Interface，高速同步通信接口（比I2C快很多），一些高性能IMU需要，4~5根线，每个设备通常需要独立 chip select。

### UART（串口通信）

UART = Universal Asynchronous Receiver/Transmitter，最经典的“串口通信”。

###  RC 协议（遥控器输入）

RC 协议负责把遥控器摇杆输入传给 PX4，PX4 再根据当前模式生成控制输出。

## 3. 更加详细的飞控内部架构

这张图是“PX4 软件内部如何把数据一步步变成电机输出”。

![5a5c2b6bc25425b7d496d7ec1dd33aa9](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/5a5c2b6bc25425b7d496d7ec1dd33aa9.png)

对应PX4源码里的一些东西：

```
commander              → src/modules/commander
navigator              → src/modules/navigator
ekf2                   → src/modules/ekf2
mc_pos_control         → src/modules/mc_pos_control
mc_att_control         → src/modules/mc_att_control
mc_rate_control        → src/modules/mc_rate_control
control_allocator      → src/modules/control_allocator
pwm_out                → src/drivers/pwm_out
mavlink                → src/modules/mavlink
logger                 → src/modules/logger
uxrce_dds_client       → src/modules/uxrce_dds_client
sensors                → src/modules/sensors
gps                    → src/drivers/gps
rc_input               → src/drivers/rc_input
```

- `ekf2` 是把传感器数据融合成稳定可信的状态估计。
- `control_allocator` 根据机型结构和电机布局，算出每个电机的输出量。
- `uxrce_dds_client` 是 PX4 和 ROS 2 之间的 uORB-topic 桥接模块。

## 4. 操作系统相关的信息

### NuttX

[NuttX](https://nuttx.apache.org//)是用于在飞行控制板上运行 PX4 的主要实时操作系统。它是开源的（BSD 许可证），轻量级、高效且非常稳定。

模块以任务的形式执行：它们拥有各自的文件描述符列表，但共享同一个地址空间。一个任务仍然可以启动一个或多个共享该文件描述符列表的线程。

每个任务/线程都有一个固定大小的栈，并且有一个周期性任务会检查所有栈是否有足够的剩余空间（基于栈着色）。

### Linux/MacOS

在 Linux 或 macOS 上，PX4 在单个进程中运行，模块在各自的线程中运行（不像 NuttX 那样区分任务和线程）。

## 5. PX4的运行流程

```
启动 PX4
  ↓
加载参数
  ↓
启动各个模块和驱动
  ↓
传感器开始发布数据
  ↓
ekf2 做状态估计
  ↓
commander 管理飞行状态
  ↓
控制器等待模式和 setpoint
  ↓
输出到电机/仿真器
```

在 SITL 里没有真实电机和真实传感器，但逻辑是一样的。只是传感器数据来自 Gazebo，电机输出也发给 Gazebo。

## 6. PX4多旋翼无人机的飞行模式

飞行模式分为**手动模式**和**自动模式**。手动模式在手动飞行（使用遥控器或操纵杆）时提供不同级别的自动驾驶辅助，而自动模式则可以完全由自动驾驶仪控制。

### 手动模式

#### Position mode：定点模式

Position mode 是最安全、最简单的手动模式，前提是飞机有可靠定位，比如 GPS 或视觉定位，或者在仿真里验证位置控制。

#### Position Slow mode：低速定点模式

仍然是位置控制模式，但是速度和偏航角速度被限制得更低。

### Altitude mode：定高模式，没有 GPS 时较安全

所以它比 Position mode 少了“水平定点”和“自动刹车”，注意水平惯性漂移。

####  Stabilized mode：自稳模式，保持姿态水平

飞机会自动回平，但不保持高度，也不保持水平位置。

#### Altitude Cruise mode：定高巡航模式

松开 roll/pitch 后，飞机不会自动回平，而是保持当前倾角继续飞。

### 自动模式

#### Hold：悬停保持

Hold 是让无人机停在当前位置和当前高度。

#### Return / RTL：返航

Return 通常也叫 RTL，Return To Launch。它需要全局位置估计，通常也就是 GPS。它不是简单地“原路返回”，而是根据 PX4 的返航逻辑和参数执行，比如返航高度、返航点、降落行为等。

#### Mission：任务模式 / 航点模式

Mission 模式会执行提前上传到飞控里的任务计划，比如提前在QGC里面画航点，通常也需要GPS。

#### Takeoff：自动起飞

Takeoff 模式让飞机自动垂直起飞，到达指定高度后通常切换到 Hold。

#### Land：立即降落

Land 模式让飞机当前位置直接执行降落。

#### Orbit：环绕飞行

Orbit 模式让飞机围绕一个中心点飞圆。

#### Follow Me：跟随模式

Follow Me 是让飞机跟随一个移动信标。这个信标可以理解成不断提供位置 setpoint 的对象，比如GPS，地面站位置，某个目标设备。

#### Offboard：外部控制模式（最重要）

无人机听从外部程序通过 MAVLink 或 ROS 2 发来的位置、速度或姿态 setpoint。

Offboard可以发送不同层级的目标：

```
位置 setpoint：飞到某个点
速度 setpoint：按某个速度飞
姿态 setpoint：保持某个姿态
```

一些限制：

```
必须持续发送 setpoint
进入 Offboard 前通常要先发送一段 setpoint
setpoint 频率太低会退出 Offboard
定位不满足时可能无法进入
丢失外部控制信号会触发 failsafe
```

## 7. NED坐标系和setpoint

### NED

NED 是 PX4 常用的“位置坐标系”，也就是North、East、Down。所以 NED 的三个坐标轴：

```
x 轴：指向北方
y 轴：指向东方
z 轴：指向地面，也就是向下
```

值position setpoint = (5, 0, -2)得注意的是：在 NED 坐标系里，**z 轴正方向是向下，不是向上。**

NED 通常是以某个参考点为原点，比如无人机上电/解锁/定位初始化时的位置。

### setpoint

在 PX4 里，setpoint 不是某一个固定东西，而是一类“你希望飞机达到的目标状态”。它可以是：

```
位置 setpoint
速度 setpoint
加速度 setpoint
姿态 setpoint
角速度 setpoint
推力 setpoint
```

最常见、最好理解的是位置 setpoint，比如：

```
position setpoint = (5, 0, -2) // 让无人机飞到起飞点前方 5 米、高度 2 米的位置
```

