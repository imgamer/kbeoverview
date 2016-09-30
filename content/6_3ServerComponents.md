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
