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
Entity在不同的space之间传送最简单的方法是简单的pop掉，比如把entity从老space移除同时在新space重新创建，在客户端和服务端都是这么简单。  
一个更复杂点的技术是，设置一个像电梯那样的在两个space上都是精确复制的封闭过渡区域。在玩家进入过渡区域后就封闭起来，开发者就能把玩家从老space pop到新space上同样区域的重复拷贝上。客户端需要转换玩家的位置到新的坐标系统，丢弃在老space中的entity认知，开始建立entity在新space中的认知。一旦客户端已经更新且新entity的位置filter也被充分填充，玩家就可以离开过渡区域（电梯门打开）继续游戏。这个过程是由CellApp自动完成的。   
bw也在开发一个传送门（gateway）的概念，这会允许在2个space间(或者长距离传送)传送不再需要等待。实现方式是在传送门(gateway)的远端维持entity的ghost，所以它们就像在附近的过渡区域一样能被发送到客户端。这个解决方案适用于公共的空间入口或者频繁的传送。这是bigworld几年前提出的概念，现在应该已经实现了。

#### 6.1.5. witness优先级队列
witness趋向于目击者的意思。  
无论何时，当有一个客户端attach到entity，为了维护它AOI中的entity列表，一个被称为witness的子对象就会关联到entity。witness建立发送到客户端的更新包，它必须优先发送最重要的信息。客户端需要最接近它的其它entity的位置和其它信息能够准确和现时。更紧密的entity应该更新的更频繁。为了达到这个目的，witness保持了在它AOI中的entity列表作为优先级队列。  
建造一个被发送到客户端的数据包，列表顶部的entity相关信息被加入到包中，直到达到需要的大小。信息包括了位置和方向，以及最后一次更新后entity所处理的事件。更多的细节请看后面的**事件历史(Event history)**。  
更接近的entity接收更频繁的更新，同时获得更大的可用带宽份额。  
引擎通过更新包的大小和更新频率来限制下行带宽。这些参数配置在res/server/kbengine_defs.xml中。

##### 6.1.5.1. 事件(event)历史
每个entity保持着最近的事件历史记录。历史记录包括它的状态改变（例如武器变化）以及是否进行一些动作（诸如跳跃或者射击）。但移动等影响volatile数据的高频动作不会被保持在历史记录列表中。  
当entity使用（consumes）它的优先级队列去构建更新包时，它查找最优先的entity（例如，最接近自身的entity）事件，添加关心的信息，这就需要为在优先级队列里每个实体的最后更新保持一个时间戳（实际上是一个序列号）。  
事件类型的历史记录是相当短的（约60秒），因为我们希望在最糟糕的情况下，优先级队列里的每个entity在30秒(大概)内都能轮到一次。  

##### 6.1.5.2. Level of detail
优先级队列为连续的信息进行数据节流。对于事件来说，我们仍需要处理有大量的事件充满可用的客户端下行带宽的情况。  
当entity的AOI中存在大量的其它entity时，互相间的信息不会被经常发送，但事件的总量还是一样的。  
为了解决这个问题，引擎使用了Level of Detail (LoD)系统处理基于事件的改变。这个原理是，越接近avatar的entity，越是需要把更多的细节发送给avatar的客户端。比如，一个玩家刚进入avatar的AoI时，avatar是不需要知道其它avatar的所有细节的。当距离100米时，仅需要看到轮廓。更精确的细节，如衣服上的徽章，在20米内就够了。  

* **无状态改变事件（Non‐state‐changing events）**  
在每个entity的类型描述中，随着每一个关联的优先级，会相应的创建所有消息的描述。  
当一个avatar添加entity的信息时，只会添加entity中大于avatar的当前所关注优先级的消息。因此，消息是否会被忽略取决于avatar和entity之间的距离。  
例如，某种聊天只能在50米内被听到，一个跳跃只能在100米内被看到。
* **状态改变事件（State‐changing events）**  
引擎无法忽略发生在范围之外的状态改变事件，因为如果entity随后足够接近，客户端得到的将是错误的状态数据，正确的还没同步过来。  
例如，只有在100米以内，客户端才关注entity手里拿着什么样的武器。如果entity在这个距离外切换了武器，这个信息不会发送给客户端，引擎只会在entity进入这个范围才会发送。  
而和非状态改变事件不同的是，如果优先级别有值的时候，状态改变事件会有开发者定义的LoDs状态的离散值。这些会被认为对客户端围绕着的环，entity类型的每个属性和这些环中的一个绑定。例如**某个entity def中的某个属性**。  
例如，假设avatar A被4个同样类型的entity围着，avatar的LoD的级别分别是20m，100m和500m。它会根据距离来处理每个entity，如下表：  

LoD消息处理。处理是yes，不处理是no：

Entity | Distance | 20m | 100m | 500m
------ | ------ | ---- | ----- | ------
  E1 | 15M | [yes] | [yes] | [yes]
  E2 | 90M | [no] | [yes] | [yes]
  E3 | 400M | [no] | [no] | [yes]
  E4 | 1000M | [no] | [no] | [no]

![图片示例]()  

当一个有witness的entity进入其它entity的LoD环圈中，从上一次它们这么接近的时候算起（since they were last this close）任何已经改变的和这个LoD环圈相关的状态被发送给客户端，引擎使用时间戳来达到这个目的，时间戳会和entity的每个属性一起记录，这个时间戳指明属性最后一次更新是什么时候。每一个在witness的AoI中的entity，当entity最后一次离开那个LoD环圈级别，对应的时间戳也会被记录。  
**（这段不准确，暂时这样）**由它本身而不允许我们去限制这个数据。为了达到这个目的，我们仅需要应用一个乘数因子到所关注的avatar级别，当这个avatar计算其它entity的时候，乘上一个小于1的因子而发送会减少数据量的影响。

#### 6.1.6. entity的脚本编程（Scripting and entities）
以下小结说明了如何对entity使用脚本。

##### 6.1.6.1. Entity类
一个entity类描述了一个entity类型。下表列出了entity的组成部分：  

Part | File
---- | -----
An XML definition file | defs/entity.def
A Python cell script | cell/entity.py
A Python base script | base/entity.py

Client需要自己解析协议并封装。  

XML定义文件被当作脚本间的接口，它定义了所有的公共方法和属性，另外还定义了全局类的全局特性：  

* entity是否需要一个volatile位置更新。
* entity如何在客户端被初始化。
* 如果有的话，entity的基类是什么。
* 如果有的话，entity的实现接口是什么。  


一个座位entity的定义文件（defs/Seat.def）如下
```xml
<root>
	<Properties>
		<seatType>
			<Type> INT8 </Type>
			<Flags> OTHER_CLIENT </Flags>
			<Default> -1 </Default> <!-- See Seat.py -->
			<DetailLevel> NEAR </DetailLevel>
		</seatType>

		<ownerID>
			<Type> INT32 </Type>
			<Flags> OTHER_CLIENT </Flags>
			<Default> 0 </Default>
		</ownerID>

		<channel>
			<Type> INT32 </Type>
			<Flags> PRIVATE </Flags>
		</channel>
	</Properties>

	<ClientMethods>
		<clientMethod1>
			<Arg> STRING </Arg> <!-- msg -->
			<DetailDistance> 30 </DetailDistance>
		</clientMethod1>
	</ClientMethods>

	<CellMethods>
		<sitDownRequest>
			<Exposed/>
			<Arg> OBJECT_ID </Arg> <!-- WHO -->
		</sitDownRequest>

		<getUpRequest>
			<Exposed/>
			<Arg> OBJECT_ID </Arg> <!-- WHO -->
		</getUpRequest>

		<ownerNotSeated>
		</ownerNotSeated>

		<tableChat>
			<Arg> STRING </Arg> <!-- msg -->
		</tableChat>
	</CellMethods>

	<LoDLevels>
		<level> 20 <hyst> 4 </hyst> <label> NEAR </label> </level>
		<level> 100 <hyst> 10 </hyst> <label> MEDIUM </label> </level>
		<level> 250 <hyst> 20 </hyst> <label> FAR </label> </level>
	</LoDLevels>
</root>
```

##### 6.1.6.3. 属性
上一节的范例文件已经说明所有的属性都被定义在XML文件中，数据类型，可选的默认值，指明如何被复制的标志(flag)，还有一个可选的LoD级别标签。  
所有的cell entity属性都需要被定义，即使是类的私有属性。这是因为CellApp需要卸载entity并发送到其它CellApp，定义所有的属性类型使得数据的传输尽可能高效。  
当python脚本修改了一个属性的值，服务端负责把这个属性的改变更新到正确的地方。  
以下用于XML文件的标志(flag)用来确定属性是如何被复制更新的：  

* ALL_CLIENT
  属性能被周围的客户端获得，包括自身。相当于同时设置了OWN_CLIENT和OTHER_CLIENT标志。
* ALL_CLIENTS
  同ALL_CLIENT
* BASE
  只能在Base上使用
* BASE_AND_CLIENT
  属性在base和客户端都可见，相当于同时设置了OWN_CLIENT和BASE标志。
* CELL_PRIVATE
  entity的内部属性。
  只在cell的entity内部可见，相当于私有属性。对于没有ghost的kbe而言，同CELL_PUBLIC.
* CELL_PUBLIC
  可以被服务端的其它entity访问。在kbe中，现在暂时和CELL_PUBLIC是一样的。
* CELL_PUBLIC_AND_OWN
  在cell上的其它entity可见，对客户端来说，仅自身客户端可见
* ...  

当一个属性值变化了，服务端会使用OWN_CLIENT和ALL_CLIENTS去检测需要同步到entity自身的客户端，使用OTHER_CLIENTS和ALL_CLIENTS去检测是否需要同步到其它客户端。  
如果一个服务端脚本修改了一个ALL_CLIENTS属性，这个改变会被复制到这个entity本身的客户端和所有其它附近的客户端，（bigworld还会更新到所有相邻的ghost）。
属性定义还包括DetailLevel标签，这用于数据的LoD处理。详细解释参见<服务端编程指南>。  

注意：当一个客户端修改了一个共享属性，这个变化不会发送给服务端，所有客户端和客户端的通信都必须通过远程方法调用。

##### 6.1.6.4. 内置属性
除XML文件中定义的属性，cell entity脚本还能访问一些cell脚本环境提供的内置属性：  
  
Built-in Property | Read/Write | Description
------- | ------- | ----------
id | Read-only | entity的整型ID
spaceID | Read-only | entity所在space的ID
vehicle | Read-only | entity的坐骑entity，没有则为None
position | Read/write | entity的位置，3个浮点数组成的元组
direction | Read/write | entity的方向，描述roll，pitch，yaw的元组
isOnGround | Read/write | 整型标志，1标示entity在地面上，0则是不在

可以通过改变entity的position属性来移动。但持续的移动，推荐使用moveToPoint方法。spaceID和vehicle属性会分别被teleport和boardVehicle方法影响。

##### 6.1.6.5. 方法
和属性类似，方法也被定义在XML文件中，client，cell和base方法被分成不同的部分。每个方法都有用名字和参数列表定义。
Cell和base方法有可选的<Exposed/>标签，用于说明此方法是否能被客户端远程调用。在cell上的Exposed方法有一个隐藏参数source，这是和调用此方法的客户端相关联entity的ID。方法定义也有一个DetailDistance标签，用于做LoD（前面说的无）。

##### 6.1.6.6. 调用客户端方法
Cell entity在其它entity的AoI中时，会在它们的客户端拥有一个相应的entity对象。有些时候cell entity想要调用所有这些client entity的方法时，有几个可用的cell脚本属性来方便调用：  

* ownClient
* otherClients
* allClients
* clientEntity  

使用这些对象调用的方法会被分发给指定的客户端。  
例如，调用附近所有entity的在客户端上的chat方法：
self.allClients.chat( "hi!" )  

##### 6.1.6.7. 内置方法
cell脚本能够访问一些内置的方法，其中一些如下：

Built-in Method | Description
------- | ------- 
destroy | 销毁entity自身
addTimer | entity所在space的ID
moveToPoint | 直线移动entity到某个点
moveToEntity | 把entity移动到另外一个entity的位置
navigate | entity向某个点移动，避免碰撞(使用寻路数据)
cancel | 撤销一个controller（timer），在kbengie中没有controller的概念
entitiesInRange | 找出指定半径内的所有指定entity


##### ~~6.1.6.8. Controllers~~
这是Bigworld引擎专有概念。kbengine是开源引擎，不考虑代码隐藏，实现上更为简单一些。  
controller是一个和cell entity相关的c++对象，它通常用于处理对脚本来说比较低效或者困难的任务，它可以有状态信息，如果entity卸载到另外一个cell，这些信息也能够通过网络传输迁移。  
当前已经实现的controller是：  

* TimerController:................提供异步的计时回调。
* MoveToPointController:..........使用给定的速度把entity移动到某个位置。

controller通常被脚本调用引发创建，例如entity.moveToPoint、entity.addTimer。任何脚本调用会创建一个controller并返回一个controller ID，这是一个唯一标识controller的整型值，使用这个ID作为参数调用entity.cancel，可以销毁controller。注意：当entity销毁后，cell会自动销毁它所有的controller。  
因为controller的处理通常是异步的，它们通过调用一个固定名字的脚本回调来通知完成的脚本对象。例如，TimerController的事件回调函数是onTimer，MoveToPointController的回调是onMove。  

##### 6.1.6.9. entity继承
一个entity类可能会从其他的entity类继承而来，会获得基类所有的属性和方法。如果.def文件使用了<Implements>标签描述了多个接口，还能添加自己的接口（interfaces）。  
甚至一个继承的类在客户端上实例化的时候，允许行为和它的基类一样。这在客户端不需要得知所有服务端实现的附加方法和属性的的情况下很有用，客户端可以把派生类 entity当作基类entity处理（在客户端上不需要有派生类的定义），在允许服务端上创建新的类派生于基类，但是客户端不需要创建新的衍生类脚本。

##### 6.1.6.10. 脚本文件范例

```python
"entity类型定义"

import KBEngine
class Seat( KBEngine.Entity ):
	def __init__( self ):
		KBEngine.Entity.__init__(self)

	# 略...
```

#### 6.1.7. 直接消息
并不是所有的事件都是通过事件历史队列发送。如果一个事件是发往一个明确的玩家或者一小撮玩家，它几乎是在它的下一个包立即分发的。  
例如，当一个玩家和一个avatar聊天，当信息到达的时候，能被立即传送到需要的avatar，avatar接着把它添加到它的下一个bundle发送给客户端。  

#### ~~6.1.8. 从ghost转发~~
在跨cell传送中收到消息如何处理，也依赖于kbengine跨地图传送的entity数据迁移是怎么做的。  

以下是适用于bigworld的策略：  
每个ghost都知道它的real cell在哪里，这样它可以把消息转发到real去处理。  
大多数消息都是拉取(pull)类型，当另一个玩家跳起来——这由avatar决定（通过优先级队列）是否和何时把这个消息发送给它的客户端。  
少数消息是推送(push)类型的，如玩家A想要和玩家B握手，要实现这个，客户端A发送一个握手请求到它的cell，接着发送一个请求到同cell的avatar B，avatarB可能是ghost，此时要转发这个请求到B的real主体，这个通信是自动和透明的。  
从ghost到real的消息转发机制也用于避免当entity改变cell时产生的同步问题。通常base entity直接发送消息给real，但在cell entity传送中时，它可能暂时通过一个ghost发送消息。  

#### 6.1.9. ~~卸载entity~~
大约每秒一次，cell会检查是否要卸载entity到其它cell。每个entity都被视为紧靠当前cell边界，边界会被人为的增加（大约10米）以避免滞后。如果entity被发现超出了当前cell的边界，它会被卸载到最适当的cell（接壤的）。  
当一个entity被卸载了，real entity对象会转换为ghost entity，反之亦然在一个entity被加载时（ghost变为real）。  

#### 6.1.10. 添加和移除cell  
当一个cell在运行时被添加到一个多cell组成的space，它会逐渐的扩张它的区域，避免entity试图改变cell时大幅波动。同样，在一处cell区域时也是缓慢逐渐减小区域知道所有的entity被移除。  
更详细，请查看**接下来CellAppMgr有关这个主题的说明**。  

#### 6.1.11. ~~负载平衡~~
cell能够被移动来调整服务端处理的负载的比例。  
基本（简单）的策略是如果一个cell和其它cell比较过载了，它的区域会被削减。相反，如果它的负荷不足，那么会增加它的区域。  

#### 6.1.12. 物理特性
因为诸如固有的延迟、大量的玩家、带宽限制等因素，实现一个让人满意的能够充分碰撞处理的游戏是比较困难的（如果不是不可能）。  
例如，考虑在人群中奔跑的情况。因为碰撞和带宽限制导致的玩家位置误差，当实际上它们在客户端上互相避开，它们的服务端位置可能在相互碰撞。即使完全碰撞，从游戏角度来看也不是一个好的做法，因为可能导致很难在人群中穿行。  
基于这些原因，BigWorld仅在客户端实现了一个entity和entity的碰撞检测系统，当各个客户端试图抢占同一个位置时，它们会被冲撞到一旁。如果碰撞的要求很严格时，潜在的扩展做法是可以发消息到服务端，例如，服务端可以检查当时的情况来改变avatar的位置/速度。  
和静态场景的碰撞，可以被服务端检查。算法能够保证玩家entity能够通过定义好的路径（portals），保证不穿墙、地板和天花板。可以设置topSpeed属性大于0来检查entity是否超速移动。具体请查看文档**<服务端编程指南>**关于proxy与玩家和entity控制的章节。

#### 6.1.13. 导航(Navigation)系统
导航系统的关键特性是：  

* 室外和室外的chunk导航。
* 室外和室内区域无缝过渡。
* 寻路网格(navpoly graphs)的动态加载。
* 因为效率做了路径缓存。  

更详细请查看文档**<服务端编程指南>**关于entity和场景，导航系统等章节。  

#### 6.1.14. 范围(Range)触发和范围查询 
当一个（具体的）entity足够接近另一个entity时，经常会有触发一个事件的需求，这个称为范围触发器。  
同样也存在在一个给定区域查找所有entity的需求，这被称为范围查询。  
为了支持这些功能，每个cell维护了它所有entity的两个列表，分别通过X和Z坐标排序。  
执行范围查询时，系统（software）只搜索其中一个列表到查询范围的距离，然后选择在这个几何范围内的entity。  
执行范围触发，对特定范围感兴趣的entity会在X和Z列表中插入高和低触发器。当entity移动，触发器和列表会被更新。当其他entity移动，列表被会更新。如果一个entity穿过触发器，或者一个触发器穿过entity，系统（software）会检查其它维度来确定这是否一个触发器事件。  
在大多数情况下验证过这个算法是非常高效。  

#### 容错(Fault tolerance)
设计时已经考虑了CellApp的容错。进程会忽然停止，或者所在机器会从网络断开，会给服务端带来小但是明显的影响。  
每个CellApp的entity会周期性的备份到它的base entity。在CellApp变成不可用时，CellAppMgr应当保证每个space仍然至少有一个cell。base entity检测到如果它们的cell entity已经丢失，那么恢复它们到它们空间的轮换cell。脚本方法可用以确保恢复的entity在游戏世界中是连续的（玩家感知不到）。  
更详细请查看文档**<服务端操作指南>的容错章节**，代码级别的实现请查看**<服务端编程指南>的容错章节**。

### 6.2. CellAppMgr
CellAppMgr负责：  

* 保持cell间正确的邻接。
* 添加新的avatar到正确的cell。
* 全局的entity ID分配。

#### 6.2.1. 注册CellApp
当一个新的CellApp启动，它通过广播消息给所有的machine守护进程来找到CelAppMgr的所在。  
如果machine进程所在机器上有一个CellAppmgr，它的地址会发回给CellApp。一旦CellApp有了CellAppMgr的地址，就能够把自己注册过去。接下来如果CellAppMgr需要创建更多的cell，新的CellApp将会作为主要的承接者。  
类似的，当CellApp停止了，在逐步的移除它的cell后，它会从CellAppMgr注销。

#### 6.2.2. ~~负载平衡~~（未实现）
引擎会动态的平衡所有space的cell负载，小的单cell space发展为大的多cell（取决于entity密度）。  
引擎使用几何方格(geometric tessellation)来把大的space分割成多个cell。当前的方格(tessellation)算法如下图：  

![Example of typical space]()

每个cell无需覆盖同样面积的space部分，但被它被分配任何负载由其CellApp支持，这可能取决于CPU负载或者entity密度。  
~~详细的cell方格规划细节暂时没有。~~

#### 6.2.3. 添加和移除cell
为了支持平缓的启动和关闭，还有可扩展性，引擎需要能够有序的添加和移除cell。为了平稳的添加一个新cell到系统中，引擎会缓慢的增加新cell的管理区域。  
首先，新cell中是没有entity的，因为初始区域什么都没有。随着cell的区域增加，entity以受控的速率迁移进来。而移除cell时是相反的过程。  
如下的例子显示了一个潜在的机制来做到这一点，与方格线相关的话在在6.2.2.  

![Inserting a new cell into the system]()

分割space是CellAppMgr的工作，基于CellApp关于它们当前负载和管理的space区域的报告。CellAppMgr使用这些信息决定是否一个CellApp应该调整它负责的区域，或者是创建新的cell还是移除一个cell。无论什么决定，都会通知给CellApp。  
除了知道自己负责的区域，每个CellApp也需要得知其它CellApp共享的相同space的基本知识（地址所负责的区域，这些知识是由CellAppMgr提供给的）。一个CellApp能够和任意数量的已存在的（通常是和相邻的）CellApp建立通信通道，来决定是卸载entity还是镜像entity的数据过去（基于entity的位置）。  

#### 6.2.4. 添加一个entity
CellAppMgr能轻易确定应该把新的entity添加到哪个cell上，因为它控制着space的方格分割。  
尽管如此，创建新的cell entity时，如果知道另一个entity在目标space，新的entity往往会绕过CellAppMgr（从而减低了它的负载）创建到它希望的space。在这种情况下，它将把自己创建到另一个已存在entity相同的CellApp上（这种创建方式更简单）。

#### 6.2.5. **多space的负载平衡**（未实现）
在多space的情况下，引擎动态和持续进行着平衡着负载，从大到小，采用一种单一的最优拟合算法。  
算法的基本目标是为每个space分配足够的处理能力，且尽量减少分布在多个CellApp（硬件机器）上处理。如果一个space必须分布在多个机器上，一个算法将被用来把space分割成多个cell。

算法是基于当前每个space的CPU需求。如下例，有5个space，其中3个请求多个CPU。这个例子，所有可用的CPU性能都是平均消耗的。  

![CPU requirement per space]()

从最大（的space）到最小的CPU消耗，引擎可能分配负载于这7个相同的CPU，如下：  

![CPU allocation per space (identical CPUs))]()

引擎能够profile CPU，针对每一个的使用进行优化。  
下图中，同样的space被分配到不同性能的CPU上（更高的长条表示性能更强大的CPU）：  

![CPU allocation per space (different CPUs)]()  

#### 6.2.6. 容错(未实现)
在出错的情况下，~~Reviver~~会重启CellAppMgr。它会找到所有的CellApp，然后查询它们的cell和相应的覆盖区域。  
从这一点上，它能保持维护cell、CellApp和space，同时能够添加新的entity到正确的cell。从存储的检查点中恢复数据或者cell维护的ID范围，它能继续扮演好作为space ID和entity ID分配的角色。  
更详细的说明，请查看<服务端操作指引>的容错章节。代码级别的CellApp容错实现，请查看<服务端编程指南>的容错章节。

### 6.3. BaseApp
BaseApp管理entity的base和proxy。proxy是base的特殊情况，代码上base是proxy的父类。  

![BaseApp managing bases and proxies]()  

#### 6.3.1. Proxies
当一个玩家登陆到服务端，对应的proxy会被创建到BaseApp，它可以决定把entity创建到某个CellApp的某个cell。BaseAppMgr会把proxy创建到负载最小的BaseApp。  
除了运行LoginApp的机器以外，运行BaseApp的机器是唯一一个需要直接和因特网连接的。Proxy给客户端一个固定的IP和端口来对接，在Proxy创建后，LoginApp会把这个IP地址返回给客户端。  
当客户端想要发送消息给它的cell entity，首先要发给proxy，然后proxy转发给cell，隔离了客户端和cell。为了实现这个，proxy需要知道隔离的enttiy所在的CellApp的地址。无论何时cell entity改变CellApp，它都是和proxy通信。  
Cell entity通过对应的proxy和它的客户端通信。这个做法的主要好处是允许proxy负责可靠的处理。其它好处包括防止CellApp直接和因特网交互，限制了客户端接收数据的地址。  
Proxy能够从多个来源收集消息只生成一个数据包再发送给客户端。这包含了proxy自己的消息，活着其它对象，例如队伍的base。取决于proxy给每个和客户端通信的消息源分配的带宽。

#### 6.3.2. Bases
Baseyou如下特性：  

1. Base能够存储entity不需要在cell间迁移的属性。  
2. cell entity（或者任何其它的服务端对象）能够和base entity交流，当它并不知道自己当前在哪个cell时。base作为一个锚点存在。  
3. 一些属性保存在base上会更高效，因为它们被“远程”对象访问（或被订阅）。  
4. Base可以是没有位置的entity，相当于控制世界事件的entity。  

Proxy是一个特殊类型的base，它有相关联的客户端并控制从它发送或者接收消息。  
像cell上的entity，base也有相关联的脚本，可以只使用脚本来实现不同的base类型。请查看demo脚本的例子。

#### 6.3.3. 容错
BaseApp支持2个方案中的一个来进行容错：  

* 分布式BaseApp备份.....每个BaseApp定期备份它的entity数据到其它（多于1个）BaseApp。  
* 非分布式BaseApp备份...BaseApp定期的备份所有entity到专用备份BaseApp。  

更多细节，查看服务器操作指南的容错章节。代码实现细节请查看服务端编程指南的容错章节。  
备份方法可以配置到 kbengine_def.xml 的｀baseAppMgr/useNewStyleBackup｀，可以查看服务端操作指南的配置说明。  

**注意**  
引擎推荐分布式BaseApp备份方式，理由如下：  

* 不需要专用的备份BaseApp
* 备份负载可以分布在一段时间内。
* 不需要IP转换，这可能不被操作系统或者路由允许。这也使得集群控制更容易。  

##### 6.3.3.1. 分布式BaseApp备份
每个BaseApp备份它的entity到多个其它的BaseApp。每个BaseApp分配一组其它的BaseApp和一个哈希函数（处理？）从entity的ID到其中一个的备份。 
经过一段时间，每个BaseApp的entity都将备份到其它常规的BaseApp。这个处理会一直进行，保证新的entity也会备份。如果一个BaseApp挂了，它的每个entity会在备份BaseApp恢复。  

![Distributed BaseApp Backup – Dead BaseApp's entities can be restored to Backup BaseApp]()

这个方法的缺点是，存在可能性，base entity之前在同一个BaseApp，之后却在不同的BaseApp（这会影响脚本编程，注意到这个情况），还有就是BaseApp出错后，连接的客户端将会断开，需要重新登录。  

##### 6.3.3.2. 非分布式BaseApp备份
这个方法需要专用BaseApp来备份正在使用的BaseApp。每个BaseApp周期性的把备份的entity数据复制到它关联的备份BaseApp。  
一个备份BaseApp能够给多个BaseApp做备份，当一个BaseApp不可用时，备份BaseApp会整个取代这个进程。  

![Non-distributed BaseApp Backup– Backup BaseApp can take the place of missing BaseApp]()

### 6.4. BaseAppMgr
#### 6.4.1. 实现(Implementation)
当一个BaseApp启动，它会把自己注册到BaseAppMgr，BaseAppMgr负责维护一个所有BaseApp的列表。

#### 6.4.2. 登录
当客户端登录，LoginApp（通过DBMrg）发送一个请求给BaseAppMgr请求添加玩家到系统。BaseAppMgr接着会添加一个proxy到负载最小的BaseApp。然后BaseAppMgr发送proxy的Ip地址给LoginApp，IP会被发送到客户端，客户端接着会使用这个IP地址和服务端进行通信。

#### 6.4.3. 容错
在出错的情况下，Reviver会重启BaseAppMgr。BaseApp会重新注册到BaseAppMgr，继续运行。  
更详细的请看关于Reviver的章节。

### 6.5. LoginApp
#### 6.5.1. 实现（Implementation）
每个LoginApp都监听一个固定的端口（可配置）等待客户端登录。  
客户端登录的详细步骤和案例在设计概念章节已经介绍。  
所有LoginApp的请求都是非阻塞的，因此它可以同时处理很多登录请求。

#### 6.5.2. 部署多个LoginApp
根据负载，单一的LoginApp应能满足大多数情况。尽管如此，还会有潜在的瓶颈，和可能的失败。引擎支持部署多个LoginApp。  
剩下的问题是如何在多个登录服务器分配客户端的登录请求。标准的方式是使用一个DNS方案类似一些流行的web服务在多个服务器平衡负载的方法。  

### 6.6. DBMgr
当前只实现了一种数据库接口组件，即MySQL。当然可以扩展其它的数据库类型，如MongoDB，Redis，你自己定制的其它类型。

#### 6.6.1. MySQL
MySQL实现了和MySQL数据库的通信，通过原生的MySQL接口。MySQL可以运行在同一台机器或者其它机器，因为协议是基于TCP的。  
每个entity类型存储在一个SQL分表中，每个类的属性是一个字段(column)，标的索引是DataBaseID。  
启动时，SQL数据库模式（schema）会检查定义entity的XML文件，确保它们存在。如果一些entity类型没有数据库表，会被自动创建。如果一些类有新的属性定义到XML文件中，但数据库中没有，字段会被自动添加到相应的表中。当模式（schema）变化很频繁时，这个功能在开发中非常实用。  


#### 6.6.2. 容错
在失败的情况下，DBMgr能够被Reviver重启（在一个不同的机器）。一旦DBMgr重启，它会通过machine进程来广播它的存在，所有其它需要的组件会开始和它通信。

MySQL数据库层也能够对一些类型的错误进行容错。如果MySQL数据库的连接丢失，它能够通过配置去连接同样的数据库，或者它的一个副本。  
~~bigworld的XML数据库是没有容错的，因为所有的数据存在RAM中直到关机。尽管DBMrg可以被Reviver重新启动，但所有在服务器上次关闭后的更新全部会丢失。~~  
DBMgr还扮演着引擎次级容错的角色。更多细节，查看<服务端编程指南>的灾难恢复章节。  

#### ~~6.6.3 BigWorld的XML数据库~~
**无需阅读。**  
The XML implementation saves data in a single XML file—`<res>/entities/db.xml`. This is good for running small servers, since it is lightweight, and does not have any dependencies on an external database.  
In this implementation, DBMgr reads the XML file on startup, and caches all player descriptions in memory. No disk I/O is performed until shutdown, when the entire XML file is written to disk.  
The XML database has several known limitations that make it unsuitable for production or serious development environments. It differs significantly to the MySQL database in the area of login processing. For more details, see the document Server Programming Guide, section User Authentication And Proxy Selection, Special notes about the XML database.  

### 6.7. Reviver（未实现）
Reviver是一个看门狗进程用来在其它进程失败的时候重启它们（因为机器的进程在运行时失败或者进程本身崩溃没反应了）。Reviver在专用的机器上启动用于做容错处理，等待其它进程连接过来。  
使用Reviver的进程如下：  

* CellAppMgr
* BaseAppMgr
* DBMgr
* LoginApp

这几个进程启动后，会在网络上搜寻Reviver，然后连接过去（如果被Revier支持的话）。Reviver周期性的ping这些进程，以检查他们是否可用。如果进程不可用（没有回复），Reviver将重启自己来替代失败的进程和机器，出错的机器/进程可以关闭服务了。  
因为Reviver在恢复一个进程后停止运行，需要多个Reviver（运行在不同的机器）。Reviver使用Machine进程来气筒新的进行。  
如果一个Reviver崩溃了，请求被管理的进程会检测到失效的ping并重新查找一个新的Reviver。这取决于确保足够的Reviver运行在合适的机器上。  
不推荐的方式是，Reviver能够通过命令行被启动，请查看<服务端操作指南>的容错章节。

### 6.8. Machine
Machine守护进程运行在服务器集群的每一台机器上，可以设置在开机时自动启动。它负责：

* 启动和停止服务端组件。
* 定位服务端组件。
* 提供机器信息统计（CPU速度，网络状态，等等）。
* 提供处理统计（内存和CPU用量）。

以下子章节描述了各个职责。

#### 6.8.1. 启动和停止服务端组件
Machine能够远程的启动和停止服务端组件。这个功能被用于服务端控制工具。  
可以使用任何用户ID来启动组件，以及使用哪个编译配置版本（Debug, Hybrid, or Release）。  
因为不是所有的用户都有自己编译的服务端，可以在每个用户的基础上指定服务端二进制文件的位置来使用。这可以在配置文件中指定。  
bigworld提供的配置文件是/etc/bwmachined.conf或/etc/bwmachined.conf。更详细的说明请查看<服务端操作指南>。  

#### 6.8.2. 定位服务端组件
无论何时一个服务端组件启动，它可以公开它的Mercury接口。Mercury将会发送一个请求注册包到本地机器的Manchine进程。  
下表描述了包中的数据域：  

Field | Description
----- | -------------
UID | User ID
PID | Progress ID
Name | Name(例如CellAppMgr)
ID | 可选的唯一ID（例如 Cell ID）  

Machine接下来会把进程添加到它的注册进程列表中。它美妙会查询/proc文件系统，检测每个进程的CPU和内存用量。  
当发布的服务端组件关闭了，它必须发送一个注销包给本地Machine。如果一个进程死亡但是没有注销，Machine会在下次poll的时候查出来并更新他们已知组件列表。  
通过发送一个查询包给manchine是可以搜寻到匹配确定域(例如名字为"CellAppMgr")服务端组件的，每个匹配的进程会发送一个应答包回复。  
例如，当一个CellApp启动，它需要找到在当前UID下运行的CellAppMgr进程地址。它发送一个广播消息到网络上的每个Machine查询，请求匹配UID的所有名字为"CellAppMgr"的进程。

#### 6.8.3. 提供机器信息统计
在启动时，Machine会检查（examines）文件/proc/cpuinfo以确定CPU的数量和速度。它也定期监视整个机器的CPU，内存和网络用量。这些信息可以通过UDP请求得到，可以被显示到一些通用的应用程序（例如web console）。  

#### 6.8.4. 提供处理性能统计
Machine会每秒检查/proc文件系统，并为每个注册的进程收集内存和CPU统计数据。这个信息可以通过UDP请求得到，可以被显示到一些通用的应用程序（例如web console）。  
