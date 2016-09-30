## 5. 设计概论
  本章讨论了引擎服务端环境的设计，对引擎组件简要概述和展示了一系列使用案例。
### 5.1 硬件组件环境
  BaseApp和LoginApp是唯一的两个需要连接到因特网的服务器硬件。  
  为了安全考虑，推荐运行BaseApp和LoginApp的机器分别有2块网卡；一个连接到因特网，另一个连接到其他的服务器群组。  

下图展示了不同组件在服务器硬件间的连接：  
  ![图片示例]()  

### 5.2. 软件组件
服务端的主要软件组件是：  

* CellApp
* CellAppMgr
* BaseApp
* BaseAppMgr
* LoginApp
* DBMgr
* ~~Reviver~~

还有一个特殊的守护进程machine，运行于每台机器。  
在生产环境，每个CellApp分别需要独占一个CPU，BaseApp也一样。所有其他进程(除了machine)可以被组织起来，根据负载情况运行在一个或多个服务器的不同组合。  
上面的硬件图上，LoginApp也是单独运行在一台机器，直接连接到因特网。DBMrg同样如此。CellAppMgr和BaseAppMgr则运行在同一台机器上(称为World server).  
  下图展示了不同服务端组件进程间的沟通路径：  
![图片示例]()  


