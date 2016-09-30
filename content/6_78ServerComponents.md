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