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

#### 5.2.1. CellApp
  每个CellApp维护着所有在其上运行的cell。  
  CellApp可以算是服务器架构中最重要的部分。每个cell维护一个庞大space的一部分或者一个cell就负责整个space。在游戏世界中，cell不会重叠，它们覆盖整个游戏空间。  

  每个cell负责维护在其边界内的entity。entity是kbengine游戏环境中最基础的操作元素，它最明显的区别特征是在space中的点坐标。~~一个cell也保持着不在它边界内但是附近entity(在附近的其它cell上)的ghost(ghost其实是entity的只读拷贝)。~~  
![CellApps managing cells]()  

#### 5.2.2. CellAppMgr
CellAppMgr的主要职责是指挥cells和CellApps。  
它来协调哪些cell运行在那些CellApp，同时调整cell的尺寸来平衡每个cell的负载。  
![CellAppMgr managing CellApps]()  

#### 5.2.3. BaseApp
在某些方面，BaseApp可以被视作服务端的防火墙。对于客户端，它们的主要目标是将其和entity在CellApp间的迁移隔离。  
每个BaseApp都包含许多base。  
每个连接上的客户端由一个功能更强大的被称为Proxy的base来对接（be served）。每个proxy只负责一个客户端，每个客户端和一个proxy对接（talk）。proxy负责把客户端发送给正确cell的消息进行重定向，base是不变的，~~cell entity则会经常迁移~~，但base总可以找到它，因此所有发送给cell的消息都是要通过base来重定向的。
通常，由BaseApp维护base。base是指一个不需要有坐标的对象或者功能。例如，一个用来实现群组聊天的对象可以是一个没有相应cell的base。
任何cell entity都能有一个相应的base entity，这样在游戏中就有一个固定的通讯锚点。
一个base被认为是一个entity的一部分。所以一个proxy是client的'base entity'，通常还会有一个相关联的'cell entity'部分，以及一个可以实例化，在任何客户端能够被看到的'client entity'。  

#### 5.2.4. BaseAppMgr
游戏中只会有一个BaseAppMgr运行，负责管理所有的BaseApp。  
它的主要工作是分配新的客户端连接给合适的BaseApp并跟踪它们。

#### 5.2.5. LoginApp
客户端和LoginApp对话以便和服务端产生一个会话控制，接着LoginApp会把玩家作为一个proxy添加到一个BaseApp上，而这个BaseApp可能会继续在一个CellApp上创建一个entity。  
每次可以运行这个组件的多个实例。

#### 5.2.6. DBMgr
DBMgr是数据库的一个接口，用于对游戏世界的状态进行持久化存储，如entity数据等。  
当一个玩家登陆，login进程会从数据库请求这个玩家entity的所有属性集合。此数据被用来在BaseApp上实例化玩家的proxy。  
当玩家登出，BaseApp会发送一个下线消息给数据库。这个消息会包含玩家entity的属性，这些属性可能已经被base或者cell entity修改。在游戏过程中也会周期性的存储这些属性。

#### ~~5.2.7. Reviver~~
Reviver是一个用于在其它进程失败时重启的看护进程。可能是其他进程所在的机器出问题了，可能是其它进程本身崩溃或者没有相应了。

#### 5.2.8. Machine
Machine是运行在服务器群组中每一个机器的守护进程。它被用于远程启动，停止和查找服务器组件。还用于监视CPU，内存和网络的使用，并把信息提供给网络用户。  
各服务器组件正是通过machine的广播消息在启动时来定位到其他组件。它的行为类似于CORBA(Common Object Request Broker Architecture:定义分布式对象系统的标准)的命名服务器(Name Server)。  
这个守护进程必须全天候运行于服务器群组的每台机器，必须作为root用户启动。具体请查看**kbengine安装指引文档**。