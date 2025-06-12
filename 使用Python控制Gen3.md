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

（通过测试）

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

