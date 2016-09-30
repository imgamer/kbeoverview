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
