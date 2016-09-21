## 5. 设计概论
  本章讨论了引擎服务端的环境设计，对引擎组件简要概述和展示了一系列使用案例。
### 5.1 硬件组件
  BaseApp和LoginApp是唯一的两个需要连接到因特网的服务器硬件。  
  为了安全考虑，推荐运行BaseApp和LoginApp的机器分别有2块网卡；一个连接到因特网，另一个连接到其他的服务器群组。  

  下图展示了不同组件在服务器硬件间的连接：
  ![图片示例]  
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
  ![图片示例]  

#### 5.2.1 CellApp
Each CellApp is responsible for all cells running on it.
CellApps are probably the most important part of the architecture. Each cell is responsible for either part of a large space or an entire one. Within the game world, cells do not overlap, and collectively they cover all game spaces.
Each cell is responsible for maintaining the entities located within its boundaries. The entity is the basic element of operations within the BigWorld game environment. Its distinguishing feature is its point position in a space. A cell may also keep ghosts (copies) of entities that are near, but outside its own boundary.
  每个CellApp维护着所有在其上运行的cell。  
  CellApp可以算是服务器体系中最重要的部分。每个cell负责一个庞大space的一部分或者是整个space。在游戏世界中，cell不会重叠，他们覆盖整个游戏空间。  

  每个cell负责维护在其边界内的entity。entity是kbengine游戏环境中最基础的操作元素，它最显著的特征是具有在space中的坐标。~~一个cell也具有不在它边界内但是附近的entity(在附近的其它cell上)的ghost(ghost其实是entity的拷贝)。~~
