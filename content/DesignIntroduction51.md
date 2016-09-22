## 5. 设计概论
  本章讨论了引擎服务端的环境设计，对引擎组件简要概述和展示了一系列使用案例。
### 5.1 硬件组件
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

#### 5.2.1. CellApp
  每个CellApp维护着所有在其上运行的cell。  
  CellApp可以算是服务器架构中最重要的部分。每个cell维护一个庞大space的一部分或者一个cell就负责整个space。在游戏世界中，cell不会重叠，它们覆盖整个游戏空间。  

  每个cell负责维护在其边界内的entity。entity是kbengine游戏环境中最基础的操作元素，它最的区别特征是在space中的点坐标。~~一个cell也保持着不在它边界内但是附近entity(在附近的其它cell上)的ghost(ghost其实是entity的只读拷贝)。~~  
![CellApps managing cells]()  

#### 5.2.2. CellAppMgr
CellAppMgr的主要职责是指挥cells和CellApps。  
它来协调哪些cell运行在那些CellApp，同时调整cell的尺寸来平衡每个cell的负载。  
![CellAppMgr managing CellApps]()  

#### 5.2.3. BaseApp
在某些方面，BaseApp可以视作服务端的防火墙。对于客户端，它们的主要目标是将其和entity在CellApp间的迁移隔离。  
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

### 5.3. 使用案例
这节描述了服务端的一些功能是如何工作的，目的是给组件在服务器上的运行以及它们是如何相互关联的作一个引导性的介绍。

#### 5.3.1. 服务器启动
启动时最重要的事情是进程间的依赖和如何让不同的进程找到彼此。  
在5.2.8 machine小节提到过，machine会运行于服务器群组的每一台机器，并负责监视运行在这台机器上的其它进程。  
当一个进程启动，它会把自己注册到本地的守护进程。这个注册包括一个接口名称。
许多进程需要知道其它进程的所在。例如，一个CellApp需要得知CellAppMgr的所在，这样才能注册过去。所以当CellApp启动，它会从machine的端口发送一个广播消息给其他所有正在监听的machine请求CellAppMgr的所在。如果其中一个machine有CellAppMgr（关联到正确的用户，服务器组件可能运行在不同的机器上，如果用户id相同则视作同一组），会把CellAppMgr的详细联系方式作为响应发送回去。

#### 5.3.2. 登陆
当一个客户端启动，它会和LoginApp通信。登陆流程如下：  

1. 客户端发送一个登陆请求（需要得知LoginApp的ip地址和端口）。
2. LoginApp监听一个固定的端口，接收到请求。
3. LoginApp把请求转发给DBMgr，以便检查登陆信息是否有效。
4. DBMgr就登陆信息查询数据库。
5. 如果登陆信息有效，DBMgr指示BaseAppMgr创建一个新的entity。
6. BaseAppMgr把entity创建请求转发给最低负载的BaseApp。
7. BaseApp创建一个新的proxy，这是一个从Base类继承而来的对象。
8. 新创建的proxy可以接着在CellApp上创建一个cell entity（看需求，例如可以等玩家在客户端选择角色后再创建）。创建方式是，proxy通过CellAppMgr发送一个请求给CellApp。
9. 最后proxy的tcp（bigworld是udp）端口会回发给客户端（通过BaseApp->BaseAppMgr->DBMgr->LoginApp->客户端，**未必准确，还需看代码确认**）。  

![Login process]()  

#### 5.3.3. 接收客户端数据
所有客户端到服务端的通信都被发送到客户端对应的proxy地址，这个地址在登陆时接收。通信采用了Mercury(bw这个协议是基于UDP的，需要和kbe确认kbe的tcp协议)机制。一旦proxy接收到数据包，(如果有相关cell entity的话)会转发给对应的CellApp，更新对应的entity数据，~~如果entity有ghost的话，更新所有对应ghost的数据~~。

#### 5.3.4. 发送数据给客户端
定期频率为10赫兹（可配置），客户端对应的cell entity会构建一个有其AOI中的所有entity充分信息的数据包，然后通过proxy把这个数据包发送给客户端。

#### 5.3.5. 镜像数据(ghosting)
**这个特性kbengine还未实现，可暂时忽略本节。**  
每个客户端对应entity维护着一个在其AOI中的其他entity的优先级队列，会有一个问题是，这些entity不会在同一个cell上。解决方案是对entity进行镜像(ghosting)。  
ghost是相邻cell上的entity的拷贝。ghost拷贝包含了其他entity通过优先级队列构造更新包时可能需要到的所有数据。如果一个entity有在其它不同cell的entity的aoi中的可能性，必须创建一个ghost到那个cell。为了实现这个目标，如果一个entity在cell边界的AOI距离（或者说ghost距离）内，引擎会在那个cell上创建这个entity的一个ghost。  
当real entity(entity的主体，非ghsot) 数据更新了，（如果有的话）也会更新给它的ghost。

#### 5.3.6. 迁移到不同的cell(Changing cells)
**在kbengine中的entity迁移只会在传送到一个新的space时发生，以下机制还未实现。**  
entity会移动，它可能完全离开了当前所在cell的范围。此时，entity已经移动到一个新的相关cell。接着会通知base它的新cell地址，以便base能够继续找到它（如果是个proxy，以便proxy能够继续正确的转发消息）。

#### 5.3.7. 负载平衡
**这个特性kbengine还未实现，可暂时忽略本节。**  
一台CellApp硬件(一般是一个CPU)和其他硬件一样，有它的负载上限。避免CellApp超载的机制是，在一般情况下，如果一个cell超载了，它会减少它的尺寸范围，卸载一些entity（会被相邻cell加载）。如果它负载轻，那么它会增加尺寸去容纳更多的entity。
