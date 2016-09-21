## 3. 术语

* **AOI(Area of Interest)**  
  entity感兴趣的物理空间区域。  
  通常情况下，这是一个围绕在entity周围的垂直方向的圆柱体，其半径为500米。  
  这个长度也被称为*ghost距离*，因为要匹配*entity*和entity将要作为模板生成ghost的cell的距离。详细参见**Real和ghost entity**的说明。  
  这个距离应设置为稍微大于一个Avatar能够看到的距离。
* **Avatar**  
  一个和游戏人物相关的*entity*。可能是一个玩家角色，或者是一个NPC(Non-player Character).
* **Base**  
  一个*object*用来表示~~entity~~不能移动的部分。  
  不像~~entity~~在cell上的部分，能够在不同的物理机器(cellapp)上移动，一个base是不能移动的，而作为一个固定的锚点(location)来和entity交流。  
  一旦创建，生命期就都会在baseapp的某个内存位置。
* **Call**  
  表示远程方法调用。与事件(*event*)不同的是call是直接在已知对象上调用。
* **Chunking**  
  这个术语用于把世界描述成很多链接在一起的块的概念。经常用在当需要说明关于渲染场景或者加载世界数据时。  
  例如，在客户端，只有当前需要被看见的chunks需要被渲染。
* **Entity**  
  一个拥有位置的对象。一个entity是cell能够维护管理的东西最通用的概念。
* **Event**  
  和call不同的是事件被发送到所注册的时间监听器，而不是已知的对象。
* **Ghost**  
  这是对非主体的real镜像对象的描述，从real对象复制而来，作为只读数据存在。主要用于实现动态负载平衡，kbengine目前还并未实现这个特性，仅仅用于传送功能。
* **Message**  
  一个message表示单个事件或者远程调用。
* **Object**  
  游戏中最通用的概念，和c++面向对象的“对象”对应。例如，一个按钮或者描述一个队伍的对象之类。
* **Packet**  
  一个packet(网络数据包)是指message的集合，它对应tcp包中发送的数据。多说一句，和kbe不同，bigworld用的是自实现的可靠udp协议。
* **Real**  
  entity对象的权威表示，至少会存在于某个cell中。目前在kbengine中所有的entity都是real。
<br/>
注：术语解释中提到的**cell**概念需要往后阅读。

