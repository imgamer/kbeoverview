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