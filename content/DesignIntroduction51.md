## 5. 设计概论
  本章讨论了引擎服务端的环境设计，对引擎组件简要概述和展示了一系列使用案例。
### 5.1 硬件组件
  BaseApp和LoginApp是唯一的两个需要连接到因特网的服务器硬件。  
  为了安全考虑，推荐运行BaseApp和LoginApp的机器分别有2块网卡；一个连接到因特网，另一个连接到其他的服务器群组。  

下图展示了不同组件在服务器硬件间的连接：  
  ![图片示例]()  

### 5.2 软件组件
服务端的主要软件组件是：  

	* CellApp
	* CellAppMgr
	* BaseApp
	* BaseAppMgr
	* LoginApp
	* DBMgr
还有一个特殊的守护进程machine，运行于每台机器。  
在生产环境，每个CellApp分别需要独占一个CPU，BaseApp也一样。所有其他进程(除了machine)可以被组织起来，根据负载情况运行在一个或多个服务器的不同组合。  
上面的硬件图上，LoginApp也是单独运行在一台机器，直接连接到因特网。DBMrg同样如此。CellAppMgr和BaseAppMgr则运行在同一台机器上(称为World server).  
  下图展示了不同服务端组件进程间的沟通路径：
![图片示例]()  

#### 5.2.1 CellApp
  每个CellApp维护着所有在其上运行的cell。  
  CellApp可以算是服务器架构中最重要的部分。每个cell维护一个庞大space的一部分或者一个cell就负责整个space。在游戏世界中，cell不会重叠，它们覆盖整个游戏空间。  

  每个cell负责维护在其边界内的entity。entity是kbengine游戏环境中最基础的操作元素，它最的区别特征是在space中的点坐标。~~一个cell也保持着不在它边界内但是附近entity(在附近的其它cell上)的ghost(ghost其实是entity的只读拷贝)。~~  
![CellApps managing cells]()  

#### 5.2.2 CellAppMgr
CellAppMgr的主要职责是指挥cells和CellApp。  
它来协调哪些cell运行在那些CellApp，同时调整cell的尺寸来平衡每个cell的负载。  
![CellAppMgr managing CellApps]()  

#### 5.2.3 BaseApp
在某些方面，BaseApp可以视作服务端的防火墙。对于客户端，它们的主要目标是将其和entity在CellApp间的迁移隔离。  
每个BaseApp都包含许多base。  
每个连接上的客户端由一个功能更强大的被称为Proxy的base来对接（be served）。每个proxy只负责一个客户端，每个客户端和一个proxy对接（talk）。proxy负责把客户端发送给正确cell的消息进行重定向，base是不变的，~~cell entity则会经常迁移~~，但base总可以找到它，因此所有发送给cell的消息都是要通过base来重定向的。
通常，由BaseApp维护base。base是指一个不需要有坐标的对象或者功能。例如，一个用来实现群组聊天的对象可以是一个没有相应cell的base。
任何cell entity都能有一个相应的base entity，这样在游戏中就有一个固定的通讯锚点。
一个base被认为是一个entity的一部分。所以一个proxy是client的'base entity'，通常还会有一个相关联的'cell entity'部分，以及一个可以实例化，在任何客户端能够被看到的'client entity'。  

#### 5.2.4 BaseAppMgr
游戏中只会有一个BaseAppMgr运行，负责管理所有的BaseApp。  
它的主要工作是分配新的客户端连接给合适的BaseApp并跟踪它们。

#### 5.2.5 LoginApp
客户端和LoginApp交流以便和服务端产生一个session。然后LoginApp

