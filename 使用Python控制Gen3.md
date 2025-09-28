# 概述

[官方 Github 项目 Kinova-kortex2_Gen3_G3L](https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L) 中存放着 使用 Python 代码控制 Kinova Gen3 的众多示例，通过这些示例 以方便学习 如何使用 Python 去控制机械臂和夹爪。

这里记录研究出来的代码怎么写的。



Kinova-kortex2_Gen3_G3L (Github) (包含 Python 和 C++ 控制机械臂的示例代码): https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L

Kortex Python API 文档：https://docs.kinovarobotics.com/ref/autogen/Messages/index.html





# Gen3 Python 开发环境配置

借助 kortex_api 包，可实现直接使用纯 Python3 代码控制 Kinova Gen3 机械臂和夹爪 (Python >= 3.5)。

Python 安装 kortex_api 包的步骤如下：

1. 下载 [kortex_api whl 文件](https://artifactory.kinovaapps.com/ui/repos/tree/General/generic-public/kortex/API/2.5.0/kortex_api-2.5.0.post6-py3-none-any.whl) (除了官方下载地址外，该笔记目录 files 文件夹下也进行了备份)

2. 使用 pip 安装下载好的 kortex_api whl 文件，命令如下：

   ```shell
   python3 -m pip install kortex_api-2.5.0.post3-py3-none-any.whl
   ```

   如果在 Ubuntu 中报错说没有 pip，使用以下命令进行安装：

   ```shell
   sudo apt-get install python3-pip
   ```

3. 这时就可以直接用 python3 运行 Github 项目中的代码了（前提还要已经用有线网络连接了计算机和机械臂，并且在计算机上配置了静态端口）

   ```shell
   python3 <example-file>.py
   ```




注意，在 Python高版本 (例如Python3.10) 执行代码时会遇到如下报错：

```shell
AttributeError: module 'collections' has no attribute 'Mapping'
```

这是因为从 Python3.3 开始，`collections.Mapping` 就已经被移动到了 `collections.abc.Mapping` 中。Python3.10 之前还仍可以使用 `collections.Mapping`，从 Python3.10 开始 `collections.Mapping` 就彻底移除，必须使用 `collections.abc.Mapping`。

修改方法是直接打开报错的文件进行修改。后续执行代码还会在其他代码位置报类似的错误，均需要修改。

在 Ubuntu 中，可以使用以下命令，用文本文档打开代码文件进行修改：

```shell
sudo gedit <从报错终端中复制的文件路径>
```



如果如果后面要卸载 kortex_api，直接使用以下命令：

```shell
python3 -m pip uninstall kortex_api
```



参考：

[官方 Github 上关于 Python 环境设置的说明]: https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L/blob/master/api_python/examples/readme.md
[关于配置 Python 环境说明的一篇 CSDN 博客]: https://blog.csdn.net/weixin_43935696/article/details/146117307

[关于处理 AttributeError: module 'collections' has no attribute 'Mapping' 报错的一篇 CSDN 博客]: https://blog.csdn.net/weixin_45471729/article/details/129982922





# 连接 gen3 的 TCP 通用方法 (精简)

```python
from kortex_api.TCPTransport import TCPTransport
from kortex_api.RouterClient import RouterClient, RouterClientSendOptions
from kortex_api.SessionManager import SessionManager
from kortex_api.autogen.messages import Session_pb2

class DeviceConnection:
    def __init__(self):
        self.ipAddress = '192.168.1.10'
        self.port = 10000
        self.credentials = ('admin', 'admin')
        self.sessionManager = None
        self.transport = TCPTransport()
        self.router = RouterClient(self.transport, RouterClient.basicErrorCallback)

    def __enter__(self):
        self.transport.connect(self.ipAddress, self.port)
        session_info = Session_pb2.CreateSessionInfo()
        session_info.username = self.credentials[0]
        session_info.password = self.credentials[1]
        session_info.session_inactivity_timeout = 10000    # 会话空闲超时（毫秒）
        session_info.connection_inactivity_timeout = 2000  # 连接空闲超时（毫秒）
        self.sessionManager = SessionManager(self.router)
        self.sessionManager.CreateSession(session_info)
        return self.router

    def __exit__(self, exc_type, exc_value, traceback):
        if self.sessionManager is not None:
            router_options = RouterClientSendOptions()
            router_options.timeout_ms = 1000 
            self.sessionManager.CloseSession(router_options)
        self.transport.disconnect()
```



使用时精简为:

```python
import os, sys
from kortex_api.autogen.client_stubs.BaseClientRpc import BaseClient  # 用于发送“非实时控制”命令，比如移动机械臂、设置模式等
from kortex_api.autogen.client_stubs.BaseCyclicClientRpc import BaseCyclicClient  # 用于实时读取状态数据（如当前位置、电流等），是“1kHz 实时反馈”的通道

sys.path.insert(0, os.path.dirname(__file__))
import gen3

def main():
    with gen3.utilities.DeviceConnection() as router:
        base = BaseClient(router)  # 创建一个 BaseClient 实例，用于与机械臂的"基本控制"服务通信
        base_cyclic = BaseCyclicClient(router)  # 创建一个 BaseCyclicClient 实例，用于访问实时反馈（位置、电压、电流等）
        feedback = base_cyclic.RefreshFeedback()

        # modules.gripper.control(base, "open")
        
if __name__ == "__main__":
    main()
```





# 连接机械臂的基础环境代码

按照官方示例文件中的做法。

将 文件 utilities.py (在 examples 文件夹下) 放在被导入的位置 (放在一个可以被公共调用的位置，以方便被多个文件调用)，以 examples 文件夹中的位置为例。

编写 Python 代码文件时，导入该文件，编写连接机械臂的基础代码环境：

```python
import sys
import os
from kortex_api.autogen.client_stubs.BaseClientRpc import BaseClient
from kortex_api.autogen.client_stubs.BaseCyclicClientRpc import BaseCyclicClient

# 这里编写控制控制机械臂的函数
# def move_to_home_position(base):
# def cartesian_action_movement(base, base_cyclic):
# ...

# 或者调用第三方文件
# from move_to_home_position import move_to_home_position
# from cartesian_action_movement import cartesian_action_movement
# ...

def main():
  sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))
  import utilities
  args = utilities.parseConnectionArguments()  
  with utilities.DeviceConnection.createTcpConnection(args) as router:
    base = BaseClient(router)
    base_cyclic = BaseCyclicClient(router)
    
    # 这里依次调用控制机械臂的函数
    # move_to_home_position(base)  
    # cartesian_action_movement(base, base_cyclic)
    # ...

if __name__ == "__main__":
  main()
```



参考：

[Github 项目 Kinova-kortex2_Gen3_G3L]: https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L/blob/master/api_python/examples/102-Movement_high_level/03-twist_command.py "api_python -> examples -> 102-Movement_high_level -> 03-twist_command.py"

(该 Github 项目在 files 文件下有备份)





# 控制机械臂回到 home 姿态

代码如下：

```python
from kortex_api.autogen.messages import Base_pb2
import threading


def check_for_end_or_abort(e):
    """Return a closure checking for END or ABORT notifications
    Arguments:
    e -- event to signal when the action is completed
        (will be set when an END or ABORT occurs)
    """
    def check(notification, e = e):
        print("EVENT : " + \
              Base_pb2.ActionEvent.Name(notification.action_event))
        if notification.action_event == Base_pb2.ACTION_END \
        or notification.action_event == Base_pb2.ACTION_ABORT:
            e.set()
    return check

  
def move_to_home_position(base):
    # Make sure the arm is in Single Level Servoing mode
    base_servo_mode = Base_pb2.ServoingModeInformation()
    base_servo_mode.servoing_mode = Base_pb2.SINGLE_LEVEL_SERVOING
    base.SetServoingMode(base_servo_mode)
    
    # Move arm to ready position
    print("Moving the arm to a safe position")
    action_type = Base_pb2.RequestedActionType()
    action_type.action_type = Base_pb2.REACH_JOINT_ANGLES
    action_list = base.ReadAllActions(action_type)
    action_handle = None
    for action in action_list.action_list:
        if action.name == "Home":
            action_handle = action.handle

    if action_handle == None:
        print("Can't reach safe position. Exiting")
        sys.exit(1)

    e = threading.Event()
    notification_handle = base.OnNotificationActionTopic(
        check_for_end_or_abort(e),
        Base_pb2.NotificationOptions()
    )
    
    base.ExecuteActionFromReference(action_handle)
    
    # Leave time to action to complete
    e.wait()
    base.Unsubscribe(notification_handle)
```



参考：

[Github 项目 Kinova-kortex2_Gen3_G3L]: https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L/blob/master/api_python/examples/102-Movement_high_level "api_python -> examples -> 102-Movement_high_level"

(该 Github 项目代码在 files 文件下有备份)





# 控制末端执行器到达指定位姿

代码如下：

```python
from kortex_api.autogen.messages import Base_pb2
import threading

def check_for_end_or_abort(e):
    """Return a closure checking for END or ABORT notifications
    Arguments:
    e -- event to signal when the action is completed
        (will be set when an END or ABORT occurs)
    """
    def check(notification, e = e):
        print("EVENT : " + \
              Base_pb2.ActionEvent.Name(notification.action_event))
        if notification.action_event == Base_pb2.ACTION_END \
        or notification.action_event == Base_pb2.ACTION_ABORT:
            e.set()
    return check

def example_cartesian_action_movement(base, base_cyclic, x, y, z, theta_x, theta_y, theta_z):    
    print("Starting Cartesian action movement ...")
    action = Base_pb2.Action()
    action.name = "Example Cartesian action movement"
    action.application_data = ""

    feedback = base_cyclic.RefreshFeedback()

    cartesian_pose = action.reach_pose.target_pose
    cartesian_pose.x = feedback.base.tool_pose_x          # (meters)  # 这里 feedback.base.tool_pose_x 是当前位姿 x，也可以直接赋予指定参数
    cartesian_pose.y = feedback.base.tool_pose_y - 0.1    # (meters)
    cartesian_pose.z = feedback.base.tool_pose_z - 0.2    # (meters)
    cartesian_pose.theta_x = feedback.base.tool_pose_theta_x # (degrees)
    cartesian_pose.theta_y = feedback.base.tool_pose_theta_y # (degrees)
    cartesian_pose.theta_z = feedback.base.tool_pose_theta_z # (degrees)

    e = threading.Event()
    notification_handle = base.OnNotificationActionTopic(
        check_for_end_or_abort(e),
        Base_pb2.NotificationOptions()
    )

    print("Executing action")
    base.ExecuteAction(action)

    print("Waiting for movement to finish ...")
    finished = e.wait(TIMEOUT_DURATION)
    base.Unsubscribe(notification_handle)

    if finished:
        print("Cartesian movement completed")
    else:
        print("Timeout on action notification wait")
    return finished
```



坐标系：以机械臂home姿态为参考，夹爪的朝向为 x 轴，从 x 轴看过去，朝左为 y 轴，朝上为 z 轴。

姿态：z 是沿 xy 轴平面旋转，x 是沿 xz 轴平面旋转，y 是沿 yz 轴平面旋转。单独修改一个，它会保持在该平面上，并在该平面上旋转。



直接赋值的话 x -180 是完全垂直朝下的，-150 朝里，-210 朝外（要给负值，正值多次执行不了），-270 就是垂直向前，甚至 -280 也能赋值

如果用 base.tool_pose 减的话，x + 30 就是上抬30度

y + 30 是从后面看过去逆时针旋转 30 度 （y 是旋转夹爪，home姿态的夹爪位置0; 90是垂直的）

z + 30 是从后面看过去向右旋转 30 度 （z 120 是向下旋转）



y 30 向后面看过去是 向左转30度，-30 则是向右转30度；0 就是最中的位置

z 是旋转夹爪，90是初始最中的位置，45 就是（从上向下看）顺时针45度



z 垂直向下时是旋转夹爪，向前时是左右转

（需要再核实一下）



参考：

[Github 项目 Kinova-kortex2_Gen3_G3L]: https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L/blob/master/api_python/examples/102-Movement_high_level/01-move_angular_and_cartesian.py "api_python -> examples -> 102-Movement_high_level -> 01-move_angular_and_cartesian.py"

(该 Github 项目在 files 文件下有备份)





# 控制夹爪张开和闭合

控制夹爪张开的代码：

```python
from kortex_api.autogen.messages import Base_pb2
import time

def control_gripper(base, mode):
  gripper_command = Base_pb2.GripperCommand()
  finger = gripper_command.gripper.finger.add()
  finger.finger_identifier = 1
  gripper_command.mode = Base_ph2.GRIPPER_SPEED
  finger.value = 0.1 if mode == "close" else -0.1
  base.SendGripperCommand(gripper_command)
  gripper_request = Base2_pb2.GripperRequest()
  
  if mode != "close":
    gripper_request.mode = Base_pb2.GRIPPER_POSITION
    while True:
      gipper_measure = base.GetMeasuredGripperMovement(gripper_request)
      if len (gripper_measure.finger):
        if gripper_measure.finger[0].value < 0.01:
          break
      else:
        break
  else:
		gripper_request.mode = Base_pb2.GRIPPER_SPEED
    while True:
      gripper_measure = self.base.GetMeasuredGripperMovement(gripper_request)
      if len (gripper_measure.finger):
        if gripper_measure.finger[0].value == 0.0:
          break
      else:
        break
    time.sleep(5)
```







# 实操记录



## 控制机械臂末端执行器到达执行位姿

```python
from kortex_api.autogen.messages import Base_pb2
import threading

TIMEOUT_DURATION = 20

def check_for_end_or_abort(e):
    def check(notification, e = e):
        print("EVENT : " + \
              Base_pb2.ActionEvent.Name(notification.action_event))
        if notification.action_event == Base_pb2.ACTION_END \
        or notification.action_event == Base_pb2.ACTION_ABORT:
            e.set()
    return check


def moveToCartesian(base, x, y, z, theta_x, theta_y, theta_z):
    action = Base_pb2.Action()
    action.name = "Example Cartesian action movement"
    action.application_data = ""

    cartesian_pose = action.reach_pose.target_pose
    cartesian_pose.x = x
    cartesian_pose.y = y
    cartesian_pose.z = z
    cartesian_pose.theta_x = theta_x
    cartesian_pose.theta_y = theta_y
    cartesian_pose.theta_z = theta_z

    e = threading.Event()
    notification_handle = base.OnNotificationActionTopic(
        check_for_end_or_abort(e),
        Base_pb2.NotificationOptions()
    )
    
    base.ExecuteAction(action)

    finished = e.wait(TIMEOUT_DURATION)
    base.Unsubscribe(notification_handle)

    if finished: print("Cartesian movement completed")
    else: print("Timeout on action notification wait")
    return finished


def moveToHome(base):
    # Make sure the arm is in Single Level Servoing mode
    base_servo_mode = Base_pb2.ServoingModeInformation()
    base_servo_mode.servoing_mode = Base_pb2.SINGLE_LEVEL_SERVOING
    base.SetServoingMode(base_servo_mode)
    
    # Move arm to ready position
    print("Moving the arm to a safe position")
    action_type = Base_pb2.RequestedActionType()
    action_type.action_type = Base_pb2.REACH_JOINT_ANGLES
    action_list = base.ReadAllActions(action_type)
    action_handle = None
    for action in action_list.action_list:
        if action.name == "Home":
            action_handle = action.handle

    if action_handle == None:
        print("Can't reach safe position. Exiting")
        return False

    e = threading.Event()
    notification_handle = base.OnNotificationActionTopic(
        check_for_end_or_abort(e),
        Base_pb2.NotificationOptions()
    )

    base.ExecuteActionFromReference(action_handle)
    finished = e.wait(TIMEOUT_DURATION)
    base.Unsubscribe(notification_handle)

    if finished:
        print("Safe position reached")
    else:
        print("Timeout on action notification wait")
    return finished


def moveToCustomize1(base, base_cyclic):
    moveToCartesian(base, base_cyclic, 0.647, -0.015, -0.053, 179.189, -0.204, 90)
```









