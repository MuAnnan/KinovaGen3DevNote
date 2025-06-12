# 概述

参考的仓库和视频：

ros2_kortex (Github) (机械臂的 ROS2 环境仓库): https://github.com/Kinovarobotics/ros2_kortex

GEN3 ROS2 Training (Youtube) (Kinova 官方使用 ROS2 开发机械臂的教程视频): https://www.youtube.com/watch?v=Vcb_A1MmC-g&list=PLz1XwEYRuku6z20TaZKKvyB2Y6fSzD5FG&ab_channel=Kinova







# ros2_kortex 仓库部署

按照 [ros2_kortex](https://github.com/Kinovarobotics/ros2_kortex) 中 README 进行部署。

注意，如果是在 Ubuntu22 ROS Humble 上部署，要选择仓库的 humble 分支 (files 文件夹下有备份)。

实际使用中，使用了源代码构建的方式进行了部署。



部署完毕后，启动实体机械臂控制节点的命令如下:

```shell
source install/setup.bash  # 在其 workspace 下
ros2 launch kortex_bringup gen3.launch.py \
  robot_ip:=192.168.1.10
```

此时，可以在另一个终端中，发布话题去控制实体机械臂。

例如，在另一个终端中，发布话题控制机械臂的各个关节:

```shell
ros2 topic pub /joint_trajectory_controller/joint_trajectory trajectory_msgs/JointTrajectory "{
  joint_names: [joint_1, joint_2, joint_3, joint_4, joint_5, joint_6, joint_7],
  points: [
    { positions: [0, 0, 0, 0, 0, 0, 0], time_from_start: { sec: 10 } },
  ]
}" -1
```



## 在 Jeston 上部署

由于 Jeston 上是 arm 芯片，由于 gazebol 不支持 arm 芯片，所以在部署时不要理会关于 gazebol 的任何内容。

另外，该仓库需要修改才能适配的 arm 

下载 linux_aarch64_gcc_7.4.zip 文件，放在任意位置，例如 `kortex/src/` 下

对 kortex_api 包的 cmake 代码进行修改:

```cmake
cmake_minimum_required(VERSION 3.14)
#include(FetchContent)
#include(ExternalProject)
#FetchContent_Declare(
#  kinova_binary_api
#  URL https://artifactory.kinovaapps.com/ui/repos/tree/General/generic-public/kortex/API/2.6.0/linux_armv7_jetson_gcc.zip
#  URL_HASH MD5=988d210bcff2009a96659c115bb4fc62
#)

#FetchContent_MakeAvailable(kinova_binary_api)
#set(FETCHCONTENT_ZIP_USE_SYSTEM ON)

set(KINOVA_BINARY_API_DIR "/home/apricity/workspace/ros2_kortex_ws/linux_aarch64_gcc_7.4")

project(kortex_api)

#Future:
# convert API_URL components from CMAKE_SYSTEM variables.
# The kinova artifactory server artifacts have slightly different expectations and they need some transformations
# Could build something like: (URL_OS)_(URL_PROCESSOR)_(?)_(URL_COMPILER).zip
# string(REPLACE "_" "-" URL_PROCESSOR ${CMAKE_SYSTEM_PROCESSOR}) # x86_64 -> x86-64
# string(TOLOWER ${CMAKE_SYSTEM_NAME} URL_OS) # Linux -> linux

#Current: only support Linux x86_64
#if(CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
#  set(API_URL https://artifactory.kinovaapps.com:443/artifactory/generic-public/kortex/API/2.5.0/linux_x86-64_x86_gcc.zip)
#elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
#  set(API_URL https://artifactory.kinovaapps.com/ui/repos/tree/General/generic-public/kortex/API/2.6.0/linux_armv7_jetson_gcc.zip)
#else()
#  set(API_URL "")
#  message(FATAL_ERROR "Unsupported System: currently support is for Linux x68_64. Detected ${CMAKE_SYSTEM_NAME} and ${CMAKE_SYSTEM_PROCESSOR}")
#endif()

#ExternalProject_Add(kinova_binary_api
#  URL ${API_URL}
#  CONFIGURE_COMMAND ""
#  BUILD_COMMAND ""
#  INSTALL_COMMAND ""
#)

find_package(ament_cmake REQUIRED)

ament_export_include_directories(include/kortex_api)

install(DIRECTORY ${KINOVA_BINARY_API_DIR}/include/client/ DESTINATION include/kortex_api)
install(DIRECTORY ${KINOVA_BINARY_API_DIR}/include/common/ DESTINATION include/kortex_api)
install(DIRECTORY ${KINOVA_BINARY_API_DIR}/include/messages/ DESTINATION include/kortex_api)
install(DIRECTORY ${KINOVA_BINARY_API_DIR}/include/client_stubs/ DESTINATION include/kortex_api)
install(DIRECTORY ${KINOVA_BINARY_API_DIR}/include/google/protobuf DESTINATION include/kortex_api/google)

ament_package()
```

kortex_driver中的cmake文件也作同等修改





参考的连接：

https://github.com/Kinovarobotics/ros2_kortex/issues/169









# MoveIt for gen3 仓库部署 & 控制实体机械臂

按照 [MoveIt Getting Started](https://moveit.picknik.ai/main/doc/tutorials/getting_started/getting_started.html) 中的指导部署 [moveit2_tutorials 仓库](https://github.com/moveit/moveit2_tutorials)。注意:

- 在 git moveit2_tutorials 仓库时，要选择 main 分支才是 gen3 机械臂的，并且在 humble 中运行是没有问题。尽管，该仓库中有 humble 分支，但是该分支下是 panda 机械臂的控制文件。
- 在最后 `colcon build` 时，要添加参数 `--parallel-workers 3` ，不然电脑很容易崩溃。



部署完毕后，使用以下命令即可启动 gen3 的 MoveIt 控制程序:

```bash
source install/setup.bash  # 在其 workspace 下
ros2 launch moveit2_tutorials demo.launch.py
```



还可以连接真实 gen3 机械臂进行控制。

打开文件 `src/moveit2_tutorials/doc/tutorials/quickstart_in_rviz/launch/demo.launch.py` ，对下面的参数进行配置:

```python
launch_arguments = {
		"robot_ip": "192.168.1.10",    # 这里声明好 IP
  	"use_fake_hardware": "false",  # 将这里声明为 false
    "gripper": "robotiq_2f_85",
    "dof": "7"
}
```

修改完毕后，使用参数 `--package-select` 仅对该 package 进行编译即可 (这样可以大大节省编译时间):

```bash
colcon build --mixin debug --packages-select moveit2_tutorials
```

注意，控制真实 gen3 机械臂，还需要启动 ros2_kortex workspace 中实体机械臂的控制节点才行。



参考的视频:

GEN3 ROS2 Training - MoveIt Quickstart in RViz (Youtube) (Kinova 官方使用 ROS2 开发机械臂的教程视频): https://www.youtube.com/watch?v=jdhAdGqqXFM&list=PLz1XwEYRuku6z20TaZKKvyB2Y6fSzD5FG&index=2&ab_channel=Kinova







# 创建 C++ 节点控制 MoveIt

以下是 [Kinova 官方的视频](https://www.youtube.com/watch?v=cEPWsTKux8c&list=PLz1XwEYRuku6z20TaZKKvyB2Y6fSzD5FG&index=3&ab_channel=Kinova) 和 [MoveIt 教程文档](https://moveit.picknik.ai/main/doc/tutorials/your_first_project/your_first_project.html) 中的描述。

在 ws_moveit 工作空间 src 目录下，创建一个节点:

```bash
ros2 pkg create \
 --build-type ament_cmake \
 --dependencies moveit_ros_planning_interface rclcpp \
 --node-name hello_moveit hello_moveit
```

打开节点中 `ws_moveit/src/hello_moveit/src/hello_moveit.cpp` 文件，删除原本的代码，写入代码:

```c++
#include <memory>
#include <rclcpp/rclcpp.hpp>
#include <moveit/move_group_interface/move_group_interface.hpp>

int main(int argc, char * argv[])
{
  // Initialize ROS and create the Node
  rclcpp::init(argc, argv);
  auto const node = std::make_shared<rclcpp::Node>(
    "hello_moveit",
    rclcpp::NodeOptions().automatically_declare_parameters_from_overrides(true)
  );

  // Create a ROS logger
  auto const logger = rclcpp::get_logger("hello_moveit");

  // Next step goes here

  // Shutdown ROS
  rclcpp::shutdown();
  return 0;
}
```

在 `// Create a ROS logger` 中写入:

```C++
// Create the MoveIt MoveGroup Interface
using moveit::planning_interface::MoveGroupInterface;
auto move_group_interface = MoveGroupInterface(node, "manipulator");

// Set a target Pose
auto const target_pose = []{
  geometry_msgs::msg::Pose msg;
  msg.orientation.w = 1.0;
  msg.position.x = 0.28;
  msg.position.y = -0.2;
  msg.position.z = 0.5;
  return msg;
}();
move_group_interface.setPoseTarget(target_pose);

// Create a plan to that target pose
auto const [success, plan] = [&move_group_interface]{
  moveit::planning_interface::MoveGroupInterface::Plan msg;
  auto const ok = static_cast<bool>(move_group_interface.plan(msg));
  return std::make_pair(ok, msg);
}();

// Execute the plan
if(success) {
  move_group_interface.execute(plan);
} else {
  RCLCPP_ERROR(logger, "Planning failed!");
}
```

对该节点进行编译，执行该文件:

```bash
colcon build --mixin debug --packages-select hello_moveit
source install/setup.bash
ros2 launch moveit2_tutorials demo.launch.py  # 首先运行该文件，启动 moveit 程序

# 在该 workspace 下，打开另一个终端
source install/setup.bash
ros2 run hello_moveit hello_moveit  # 运行该 C++ 节点，该节点控制 moveit，规划 gen3 到达指定位置
```





完整的代码为:

```c++
#include <memory>
#include <rclcpp/rclcpp.hpp>
#include <moveit/move_group_interface/move_group_interface.hpp>

int main(int argc, char * argv[]) {
  rclcpp::init(argc, argv);
  auto const node = std::make_shared<rclcpp::Node>(
		"hello_moveit",
    rclcpp::NodeOptions().automatically_declare_parameters_from_overrides(true)
  );

  auto const logger = rclcpp::get_logger("hello_moveit");

  // Next step goes here
  // Create the MoveIt MoveGroup Interface
  using moveit::planning_interface::MoveGroupInterface;
  auto move_group_interface = MoveGroupInterface(node, "manipulator");

  // Set a target Pose
  auto const target_pose = []{
    geometry_msgs::msg::Pose msg;
    msg.orientation.w = 1.0;
    msg.position.x = 0.28;
    msg.position.y = -0.2;
    msg.position.z = 0.5;
    return msg;
  }();
  move_group_interface.setPoseTarget(target_pose);

  // Create a plan to that target pose
  auto const [success, plan] = [&move_group_interface]{
    moveit::planning_interface::MoveGroupInterface::Plan msg;
    auto const ok = static_cast<bool>(move_group_interface.plan(msg));
    return std::make_pair(ok, msg);
  }();

  // Execute the plan
  if(success) {
    move_group_interface.execute(plan);
  } else {
    RCLCPP_ERROR(logger, "Planning failed!");
  }

  // Shutdown ROS
  rclcpp::shutdown();
  return 0;
}
```





保持 moveit 的工作空间不变。

开辟一个新的工作空间，写两个 package ，一个 c++ 的服务端节点，一个 

这里用到了 moveit_ros_planning_interface 的依赖，可以创建 package 时自动添加(看一眼自动添加 package.xml 和 CMakeLists.txt 内容是啥样的)，也可以手动添加

package.xml:

```xml
<depend>moveit_ros_planning_interface</depend>
<build_depend>moveit_ros_planning_interface</build_depend>  // 下面这两行不知道能不能添加  // 创建时就写的话就只有 depend 一个
<exec_depend>moveit_ros_planning_interface</exec_depend>
```

CMakeLists.txt:

```
find_package(moveit_ros_planning_interface REQUIRED)

ament_target_dependencies(your_target
  rclcpp
  moveit_ros_planning_interface
  # 其他依赖...
)
```

在使用时，这两个 workspaces 都要 source







# 自创建 C++ 服务端和 Python 客户端实现在 Python 中控制并集成开发

流程如下：

1. 创建一个 workspace。

2. 在 workspace 下的 src 文件夹中，初始化存放 C++ 服务端的 package:

   ```bash
   ros2 pkg create \
   	--build-type ament_cmake \
   	--dependencies rclcpp moveit_ros_planning_interface geometry_msgs\
   	--node-name moveit_service_node \
   	moveit_service_package
   ```

   1. 在该 package 下，创建 `srv/SetPose.srv` 文件，写入:

      ```
      geometry_msgs/Pose target_pose
      ---
      bool success
      string message
      ```

      打开 `CMakeLists.txt` 文件，加入以下内容:

      ```cmake
      find_package(rosidl_default_generators REQUIRED)
      
      rosidl_generate_interfaces(${PROJECT_NAME}
        "srv/SetPose.srv"
        DEPENDENCIES geometry_msgs
      )
      
      # 因为还要在自己的 package node 中使用该 srv，所以还要加入下面两行内容
      rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} rosidl_typesupport_cpp)
      target_link_libraries(moveit_service_package "${cpp_typesupport_target}")
      
      ament_export_dependencies(rosidl_default_runtime)
      ```

      打开 `package.xml` 文件，加入以下内容:

      ```xml
      <buildtool_depend>rosidl_default_generators</buildtool_depend>
      <exec_depend>rosidl_default_runtime</exec_depend>
      <member_of_group>rosidl_interface_packages</member_of_group>
      ```

   2. 打开 `moveit_service_node.cpp` 文件，写入:

      ```cpp
      #include <rclcpp/rclcpp.hpp>
      #include <memory>
      #include <moveit/move_group_interface/move_group_interface.hpp>
      #include "moveit_server_package_cpp/srv/set_pose.hpp"
      
      using std::placeholders::_1;
      using std::placeholders::_2;
      
      std::shared_ptr<rclcpp::Node> node;
      std::shared_ptr<moveit::planning_interface::MoveGroupInterface> move_group_interface;
      
      void handle_set_pose(
          const std::shared_ptr<moveit_server_package_cpp::srv::SetPose::Request> request,
          std::shared_ptr<moveit_server_package_cpp::srv::SetPose::Response> response)
      {
          RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "接收到目标位姿态请求");
          move_group_interface->setPoseTarget(request->target_pose);
          moveit::planning_interface::MoveGroupInterface::Plan plan;
          bool success = static_cast<bool>(move_group_interface->plan(plan));
      
          if (success)
          {
              move_group_interface->execute(plan);
              response->success = true;
              response->message = "行动执行成功";
              RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "行动执行成功");
          }
          else
          {
              response->success = false;
              response->message = "行动执行失败";
              RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "规划失败");
          }
      }
      
      int main(int argc, char *argv[])
      {
          rclcpp::init(argc, argv);
          node = std::make_shared<rclcpp::Node>(
              "moveit_server_node",
              rclcpp::NodeOptions().automatically_declare_parameters_from_overrides(true));
          move_group_interface = std::make_shared<moveit::planning_interface::MoveGroupInterface>(node, "manipulator");
          move_group_interface -> setMaxVelocityScalingFactor(0.8);  // 该语句可以控制机械臂移动的速度
          auto service = node->create_service<moveit_server_package_cpp::srv::SetPose>("set_target_pose", &handle_set_pose);
          RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "MoveIt 服务端已启动，等待请求...");
          rclcpp::spin(node);
          rclcpp::shutdown();
          return 0;
      }
      ```

3. 此时如果编译了工作空间，就可以使用以下命令控制:

   ```bash
   ros2 service call /set_target_pose moveit_server_package_cpp/srv/SetPose "{target_pose: {position: {x: 0.3, y: -0.2, z: 0.6}, orientation: {w: 1.0}}}"
   ```

4. 在 workspace 文件夹下的 src 文件夹中，初始化存放 Python 客户端的 package:

   ```bash
   ros2 pkg create \
   	--build-type ament_python \
   	--dependencies rclpy moveit_service_package \
   	--node-name moveit_client_node \
   	moveit_client_package
   ```

   1. 打开 `moveit_client_node.py` 文件，写入:

      ```python
      import rclpy
      from rclpy.node import Node
      from geometry_msgs.msg import Pose
      from your_package_name.srv import SetPose  # 替换为你的实际包名
      
      class MoveItClient(Node):
          def __init__(self):
              super().__init__('moveit_client')
              self.cli = self.create_client(SetPose, 'set_target_pose')
      
              while not self.cli.wait_for_service(timeout_sec=1.0):
                  self.get_logger().info('服务未响应，等待中...')
      
              self.req = SetPose.Request()
      
          def send_request(self):
              # 设置目标位姿
              pose = Pose()
              pose.position.x = 0.28
              pose.position.y = -0.2
              pose.position.z = 0.5
              pose.orientation.w = 1.0  # 默认单位四元数
      
              self.req.target_pose = pose
              future = self.cli.call_async(self.req)
              rclpy.spin_until_future_complete(self, future)
      
              if future.result() is not None:
                  self.get_logger().info(f'响应: success={future.result().success}, message="{future.result().message}"')
              else:
                  self.get_logger().error('调用服务失败')
      
      def main(args=None):
          rclpy.init(args=args)
          client = MoveItClient()
          client.send_request()
          client.destroy_node()
          rclpy.shutdown()
      
      if __name__ == '__main__':
          main()
      ```

      





# 固定角转四元数

```python
from scipy.spatial.transform import Rotation as R

# 假设你的 Fixed Angles 是绕世界轴的 XYZ 顺序，单位是度
x_angle = 20  # degrees
y_angle = 40
z_angle = 20

# 创建 Rotation 对象（注意 'xyz' 是固定角顺序）
# 大写的 'XYZ' 是全局参考系，小写的 'xyz' 是物体参考系
r = R.from_euler('XYZ', [x_angle, y_angle, z_angle], degrees=True)

# 转换为四元数（四元数顺序为 [x, y, z, w]）
quat = r.as_quat()

print("Quaternion (x, y, z, w):", quat)
```



scipy 中 from_euler 的文档:
https://docs.scipy.org.cn/doc/scipy/reference/generated/scipy.spatial.transform.Rotation.from_euler.html