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

