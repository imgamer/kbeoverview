## 6. 服务端组件  
本章描述了引擎服务端各种组件和它们是如何一起工作去支持一个游戏环境的。  

### 6.1. CellApp
#### 6.1.1. CellApp和cell
CellApp是运行在CellApp机器（可以不是一台独立的计算机而是一个CPU）上的一个进程或者说一个可执行程序。  
CellApp管理大量Cell，或者说是Cell类的实例。一个cell覆盖着整个space区域。  
CellApp机器的目标是只运行一个machine和每个CPU一个CellApp，而不应该再有别的进程。

~~对于bigworld引擎，一个Cell是一个庞大space的一块物理部分，或一个小型space的全部。为了均匀的分担负载，bw~~把大型space分割成许多cell。分配策略是，通常会把一个大型的cell分配给每个CellApp，当处理小型space时，例如动态创建的副本space，引擎通常每个CellApp分配几个。

#### 6.1.2. Entities
CellApp关注entity这样泛化的类型。每个CellApp维护一个real列表和在其范围内的ghost，CellApp也维护一个需要定期更新（比如10次/秒）的entity列表，这些entity则维护着AOI。  
每个entity能理解和它的类型联系在一起的消息（def方法）。CellApp分发消息给恰当的entity，由entity自己去解释。实现这个的方法是通过拥有每一个实体，包括一个指向entity类型对象的指针。这个对象拥有和entity类型关联的信息，这样这个类型的消息就能够被处理且和脚本关联起来。  
每个entity类型（定义在entities.xml中）有它自己的脚本文件。脚本在一个特定实体的环境中执行（比如一个this指针或者python中的self），并有权访问其它entity有选择提供的（服务端或者客户端数据），以及每个entity的基本成员（例如id，position这些内建属性）的数据。  
脚本负责处理发送给entity的消息。  
当任何服务端或者客户端数据变化了，这个改变会被自动转发给每个感兴趣的entity。


#### 6.1.3. Real and ghost entities
**在kbe中几乎所有的entity都是Real entity，只有在传送时会涉及到ghost entity。本节可略过。**  
<br/>
为了高效的更新客户端，每个avatar在它的AOI中保持着一个entity列表，通常是一个500（可配置）米半径的圆圈。一个问题是AOI中不是所有的entity都在同一个cell。  
![Entity's AoI of crossing cell boundaries]()

解决方案是做镜像。ghost是附近cell上的entity的拷贝。ghost拷贝包含了其他entity通过优先级队列构造更新包时所需的所有数据。  
尽管如此，每个entity独自确定哪些entity是在临近的cell需要被镜像是非常低效的。这个的解决方案是一般有cell来管理ghost，如果一个entity在一个cell的ghost距离内，cell会创建一个这个entity的ghost到附近的这个cell。
real entity在术语表中被引入以区分enityt的主体和ghost。下图显示了cell 1维护的所有entity。  
![Cell 1 maintains ghosts of entities controlled by adjacent cells]()  

每秒1次，cell通过他的real entity检查是否应该从它临近的cell添加或者删除ghost。通过发送消息给临近cell来告诉它们增减ghost。  
![Cell 1 ensures that adjacent cells maintain ghosts of its real entities]()  

#### 6.1.4. 在space间传送
The simplest method to transition between different spaces is simply to ʹpopʹ, i.e., to remove the entity from the old space and recreate it in the new one. This is simple on both client and server.
A somewhat more sophisticated technique is to have an enclosed transition area such as a lift, which is precisely duplicated in both spaces. After the player enters the transition area, and it becomes enclosed, the developer can ʹpopʹ the player out of the old space and into the duplicate copy of the area in the new space. The client needs to transform the playerʹs position into the new coordinate system, discard knowledge of entities in the old space, and start building up knowledge of entities in the new space. Once the client has been updated and the position filters of the new entities are sufficiently filled, the player can leave the transition area (lift door opens) and continue. Much of this is automated by the CellApp.
BigWorld has also been developing the concept of a gateway, which would allow transitions between spaces (or long‐range teleportation) without needing to wait. This would be done by maintaining ghosts of entities on the far side of a gateway, so that they can be sent to the client as it approaches the transition area. This solution would be suitable for public portals, or for frequently made transitions. Please contact BigWorld support if you wish to use gateways.
在不同的space之间传送最简单的方法是简单的pop掉，比如把entity从老space移除同时在新space重新创建，在客户端和服务端都是这么简单。