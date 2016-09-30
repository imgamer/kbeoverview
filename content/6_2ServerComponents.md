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
