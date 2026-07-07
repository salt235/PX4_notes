# ROS1入门仿真demo代码分析

## 代码注释分析

```cpp
/**
 * @file offb_node.cpp
 * @brief Offboard control example node, written with MAVROS version 0.19.x, PX4 Flight
 * Stack and tested in Gazebo Classic SITL
 */

#include <ros/ros.h>
#include <geometry_msgs/PoseStamped.h>
#include <mavros_msgs/CommandBool.h>
#include <mavros_msgs/SetMode.h>
#include <mavros_msgs/State.h>


mavros_msgs::State current_state; // 这个是ros创建的state变量，用来接收px4的state
// 这个回调函数，就是用来不断接收状态的
void state_cb(const mavros_msgs::State::ConstPtr& msg){
    current_state = *msg;
}

int main(int argc, char **argv)
{
    ros::init(argc, argv, "offb_node");
    ros::NodeHandle nh; // 句柄，用来创建topic和service

    // 监听 /mavros/state 这个话题，一旦有新消息，就调用 state_cb() 函数
    ros::Subscriber state_sub = nh.subscribe<mavros_msgs::State>
            ("mavros/state", 10, state_cb);

    // 这个发布者是向PX4发布目标位置的
    ros::Publisher local_pos_pub = nh.advertise<geometry_msgs::PoseStamped>
            ("mavros/setpoint_position/local", 10);

    // 在这个demo中，MAVROS 是 Service Server，offb_node 是 Service Client
    // 创建了一个 service 客户端，准备以后调用 /mavros/cmd/arming 这个服务
    ros::ServiceClient arming_client = nh.serviceClient<mavros_msgs::CommandBool>
            ("mavros/cmd/arming");
    
    // 创建了一个 service 客户端，准备以后调用 mavros/set_mode 这个服务
    ros::ServiceClient set_mode_client = nh.serviceClient<mavros_msgs::SetMode>
            ("mavros/set_mode");

    // 创建一个循环频率控制器，频率是 20Hz，频率必须大于 2Hz
    // 后面调用 rate.sleep()，就会自动控制睡眠时间，使的整个while每秒循环20次
    ros::Rate rate(20.0);

    // 等待飞控连接
    while(ros::ok() && !current_state.connected){
        ros::spinOnce(); //处理一次 ROS 回调，会调用 state_cb() 检查 /mavros/state
        rate.sleep();
    }

    // 创建目标点
    geometry_msgs::PoseStamped pose; // PoseStamper 是带时间戳的位姿
    pose.pose.position.x = 0;
    pose.pose.position.y = 0;
    pose.pose.position.z = 2;

    // 正式开始前，预先发送 setpoint
    // 也就是说，在正式切换到 OFFBOARD 之前，先连续发送 100 次目标点
    // 按照之前设定的 20Hz，大概会进行 5s
    // 因为 PX4 有一个安全机制
    // 你不能突然切到 OFFBOARD，然后才开始发目标点。PX4会要求在进入 OFFBOARD 之前，已经收到稳定的 setpoint 数据流。
    for(int i = 100; ros::ok() && i > 0; --i){
        local_pos_pub.publish(pose);
        ros::spinOnce();
        rate.sleep();
    }


    // 下面这两个 mavros_msgs 是等会要用到的请求数据，先创建好
    mavros_msgs::SetMode offb_set_mode;
    offb_set_mode.request.custom_mode = "OFFBOARD";
    mavros_msgs::CommandBool arm_cmd;
    arm_cmd.request.value = true;

    // 时间变量 last_request，记录“上一次发送服务请求的时间”
    ros::Time last_request = ros::Time::now();

    // 主循环
    while(ros::ok()){
        // 只有超过 5 秒，才允许再次请求切换 OFFBOARD 和解锁
        if( current_state.mode != "OFFBOARD" &&
            (ros::Time::now() - last_request > ros::Duration(5.0))){
            // 用之前创建的 serviceClient 来调用 mavros 的模式切换服务
            if( set_mode_client.call(offb_set_mode) &&
                offb_set_mode.response.mode_sent){
                ROS_INFO("Offboard enabled");
            }
            last_request = ros::Time::now();
        } else {
            // 用之前创建的 serviceClient 来调用 mavros 的电机解锁服务
            if( !current_state.armed &&
                (ros::Time::now() - last_request > ros::Duration(5.0))){
                if( arming_client.call(arm_cmd) &&
                    arm_cmd.response.success){
                    ROS_INFO("Vehicle armed");
                }
                last_request = ros::Time::now();
            }
        }

        // 无论现在有没有切成功 OFFBOARD，无论有没有解锁，每轮循环它都会一直发布目标位置
        local_pos_pub.publish(pose);

        // 回调，不断检查 current_state
        ros::spinOnce();
        rate.sleep();
    }

    return 0;
}

```

## 一些小问题解释

### Topic 和 Service

在 ROS 里，`Topic` 是持续通信，适合“状态、传感器数据、控制量”这种不断变化的数据。`Service` 是一次性请求，可以理解为接收方调用发布方的函数，适合“解锁、切模式、开始任务、保存地图”这种明确动作。

在这个demo里面：

```
关于 Topic：
    MAVROS 发布 /mavros/state
    offb_node 订阅 /mavros/state

    offb_node 发布 /mavros/setpoint_position/local
    MAVROS 订阅 /mavros/setpoint_position/local
    
关于 Service：
    MAVROS 是 ServiceServer
    offb_node 是 ServiceClient
```

其中涉及到的所有话题名，都是 mavros 那边规定好的，所以要研究一下相关的 API。

### 关于 MAVROS 的相关话题

- /mavros/state

  消息类型：mavros_msgs/State
  发布者：MAVROS
  订阅者：自己的控制节点
  作用：读取 PX4 当前连接状态、模式、解锁状态

  ```cpp
  current_state.connected
  current_state.mode
  current_state.armed
  ```

- /mavros/setpoint_position/local

  消息类型：geometry_msgs/PoseStamped
  发布者：自己的控制节点
  订阅者：MAVROS
  作用：给 PX4 发送本地位置目标点
  
  ```cpp
  geometry_msgs::PoseStamped pose;
  // PoseStamped 是带时间戳的位姿信息
  pose.pose.position.x = 0;
  pose.pose.position.y = 0;
  pose.pose.position.z = 2;
  // pose.pose.orient
  ```

- /mavros/cmd/arming

  服务类型：mavros_msgs/CommandBool
  Service Server：MAVROS
  Service Client：自己的控制节点
  作用：请求 PX4 解锁或上锁

  ```cpp
  // 创建 Clinet
  ros::ServiceClient arming_client = nh.serviceClient<mavros_msgs::CommandBool>("mavros/cmd/arming");
  // 准备请求数据
  mavros_msgs::CommandBool arm_cmd;
  arm_cmd.request.value = true;
  // 调用服务
  arming_client.call(arm_cmd);
  ```
  
- /mavros/set_mode

  服务类型：mavros_msgs/SetMode
  Service Server：MAVROS
  Service Client：你的控制节点
  作用：请求 PX4 切换飞行模式

  ```cpp
  // 创建 Client
  ros::ServiceClinet set_mode_client = nh.serviceClinet<mavros_msgs::SetMode>("mavros/set_mode");
  // 准备请求数据
  mavros_msgs::SetMode offb_set_mode;
  offb_set_mode.request.custom_mode = "OFFBOARD";
  // 调用服务
  set_mode_client.call(offb_set_mode);
  ```
### 关于 geometry_msgs

它不是 MAVROS 专属的，而是 ROS 通用的基础几何消息包。

```cpp
// 常见的一些api
geometry_msgs::PoseStamped // 带时间戳的位姿
geometry_msgs::TwistStamped // 带时间戳的速度
geometry_msgs::Point
geometry_msgs::Quaternion
geometry_msgs::Vector3
```

关于 PoseStampd 的例子：

```cpp
// 位置
pose.pose.position.x = 0;
pose.pose.position.y = 0;
pose.pose.position.z = 2;

// 四元数姿态，控制朝向，之后再研究
pose.pose.orientation.x
pose.pose.orientation.y
pose.pose.orientation.z
pose.pose.orientation.w
```

