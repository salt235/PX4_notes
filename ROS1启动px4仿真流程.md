# ROS1启动px4仿真流程

前提是把MAVROS安装好，参考[官方的教程](https://docs.px4.io/main/en/ros/mavros_installation)，我没有用源码安装。

## 开第一个终端

```bash
cd PX4-Autopilot/
make px4_sitl_default gazebo-classic
# 启动 PX4 + Gazebo
```

系统内部发生：

- 启动PX4飞控
- Gazebo启动
- MAVLink建立

## 开第二个终端

```bash
roslaunch mavros px4.launch fcu_url:=udp://:14540@127.0.0.1:14557
```

MAVROS 发生：

-  连接 PX4
- MAVLink → ROS 映射
- ROS → MAVLink 控制通道

## 运行一个最小demo

### 1. 创建工作空间

   ```bash
   mkdir -p ~/catkin_ws/src
   cd ~/catkin_ws
   catkin_make
   ```
### 2. 初始化环境

```bash
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 3. 创建 ROS package 和 C++节点

```bash
cd ~/catkin_ws/src
catkin_create_pkg offboard_demo roscpp geometry_msgs mavros_msgs

cd offboard_demo
mkdir src

cd src
touch offb_node.cpp
```

然后把这一段demo代码粘贴进 `offb_node.cpp`：

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

mavros_msgs::State current_state;
void state_cb(const mavros_msgs::State::ConstPtr& msg){
    current_state = *msg;
}

int main(int argc, char **argv)
{
    ros::init(argc, argv, "offb_node");
    ros::NodeHandle nh;

    ros::Subscriber state_sub = nh.subscribe<mavros_msgs::State>
            ("mavros/state", 10, state_cb);
    ros::Publisher local_pos_pub = nh.advertise<geometry_msgs::PoseStamped>
            ("mavros/setpoint_position/local", 10);
    ros::ServiceClient arming_client = nh.serviceClient<mavros_msgs::CommandBool>
            ("mavros/cmd/arming");
    ros::ServiceClient set_mode_client = nh.serviceClient<mavros_msgs::SetMode>
            ("mavros/set_mode");

    //the setpoint publishing rate MUST be faster than 2Hz
    ros::Rate rate(20.0);

    // wait for FCU connection
    while(ros::ok() && !current_state.connected){
        ros::spinOnce();
        rate.sleep();
    }

    geometry_msgs::PoseStamped pose;
    pose.pose.position.x = 0;
    pose.pose.position.y = 0;
    pose.pose.position.z = 2;

    //send a few setpoints before starting
    for(int i = 100; ros::ok() && i > 0; --i){
        local_pos_pub.publish(pose);
        ros::spinOnce();
        rate.sleep();
    }

    mavros_msgs::SetMode offb_set_mode;
    offb_set_mode.request.custom_mode = "OFFBOARD";

    mavros_msgs::CommandBool arm_cmd;
    arm_cmd.request.value = true;

    ros::Time last_request = ros::Time::now();

    while(ros::ok()){
        if( current_state.mode != "OFFBOARD" &&
            (ros::Time::now() - last_request > ros::Duration(5.0))){
            if( set_mode_client.call(offb_set_mode) &&
                offb_set_mode.response.mode_sent){
                ROS_INFO("Offboard enabled");
            }
            last_request = ros::Time::now();
        } else {
            if( !current_state.armed &&
                (ros::Time::now() - last_request > ros::Duration(5.0))){
                if( arming_client.call(arm_cmd) &&
                    arm_cmd.response.success){
                    ROS_INFO("Vehicle armed");
                }
                last_request = ros::Time::now();
            }
        }

        local_pos_pub.publish(pose);

        ros::spinOnce();
        rate.sleep();
    }

    return 0;
}
```

接着修改 CMakeLists.txt（关键！！！）：

```bash
nano ~/catkin_ws/src/offboard_demo/CMakeLists.txt
# 在最后加上
add_executable(offb_node src/offb_node.cpp)

target_link_libraries(offb_node
  ${catkin_LIBRARIES}
)

add_dependencies(offb_node ${catkin_EXPORTED_TARGETS})
```

### 4. 编译

```bash
cd ~/catkin_ws
catkin_make
```

### 5. 运行

```bash
rosrun offboard_demo offb_node
```

### 6. 启动PX4 和 MAVROS

```bash
# 第一个终端
cd PX4-Autopilot
make px4_sitl_default gazebo-classic
# 第二个终端
roslaunch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557"
```

### 7. 运行节点

```bash
source ~/catkin_ws/devel/setup.bash
rosrun offboard_demo offb_node
```

### 8. 结果分析

无人机会缓慢飞到上空2m处。