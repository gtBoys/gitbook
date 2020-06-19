# ZooKeeper

> Zookeeper是一个开放源码的分布式协调服务，它是集群的管理者，监视着集群中各个节点的状态根据节点提交的反馈进行下一步合理操作。最终，将简单易用的接口和性能高效、功能稳定的系统提供给用户。
>
> 分布式应用程序可以基于ZooKeeper的实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等功能。



#### Zookeeper保证了如下分布式一致性特性

1. 顺序一致性
2. 原子性
3. 单一视图
4. 可靠性
5. 实时性（最终一致性）

客户端的读请求可以被集群中的任意一台机器处理，**如果读请求在节点上注册了监听器，这个监听器也是由所连接的zookeeper机器来处理。**对于写请求，这些请求会同时发给其他zookeeper机器并且达成一致后，请求才会返回成功。因此，随着zookeeper的集群机器增多，读请求的吞吐会提高但是写请求的吞吐会下降。

有序性zookeeper中非常重要的一个特性，所有的更新都是全局有序的，每个更新都有一个唯一的时间戳，这个时间戳成为zxid(Zookeeper Transaction Id).而读请求只会相对于更新有序，也就是读请求的返回结果中会带这个zookeeper最新的zxid.



#### Zookeeper 提供了什么

1. 文件系统
2. 通知机制





#### Zookeeper文件系统

Zookeeper提供一个多层级的节点命名空间（节点称为znode）与文件系统不同的是，这些节点都可以设置关联的数据，文件系统中只有文件节点可以存放数据而目录节点不行。

Zookeeper为了保证高吞吐和低延迟，在内存中维护了这些树状的目录结构，**这种特性使得Zookeeper不能用于存放大量的数据，每个节点的存放数据上限为1M。**





#### ZAB协议

ZAB协议是为分布式协调服务Zookeeper专门设计的一种支持崩溃恢复的原子广播协议。

ZAB协议包括两种基本的模式：**崩溃恢复和消息广播。**

当整个zookeeper集群刚刚启动或者Leader服务器宕机、重启或者网络故障导致不存在过半的服务器与Leader服务器保持正常通信时，所有进程（服务器）进入奔溃恢复模式，首先选举产生新的Leader服务器，然后集群中Follower服务器开始与新的Leader服务器进行数据同步，当集群中超过半数机器与该Leader服务器完成数据同步之后，退出恢复模式进入消息广播模式，Leader服务器开始接受客户端的事务请求生成事物提案来进行事务请求处理。





#### 四种类型的数据节点Znode

1. **PERSISTENT**-持久节点

   除非手动删除，否则节点一直存在于Zooleeper上

2. **EPHEMERRAL**-临时节点

   临时节点的生命周期于客户端会话绑定，一旦客户端会话失败（客户端于Zookeeper链接断开不一定会话失败），那么这个客户端创建的所有临时节点都会被移除。

3. **PERSISTENT_SEQUENTIAL**-持久顺序节点

   基本特性同持久节点，只是增加了顺序属性，节点名后边会追加一个由父节点维护的自增整型数字。

4. **EPHEMERAL_SEQUENTIAL**-临时顺序节点

   借本特性同临时节点，增加了顺序属性，节点名后边会追加一个由父节点维护的自增整型数字。





#### Zookeeper Watcher 机制 -- 数据变更通知

Zookerper允许客户端向服务端的某个Znode注册一个Watcher监听，当服务端的一些指定事件触发了这个Watcher，服务端会向指定客户端发送一个事件通知来实现分布式的通知功能，然后客户端根据Watcher通知状态和事件类型做出业务上的改变。



**工作机制**

1. 客户端注册Watcher
2. 服务端处理Watcher
3. 客户端回调Watcher



**Watcher特性总结**

1. 一次性

   无论是服务端还是客户端，一旦一个Watcher被触发，Zookeeper都将其从相应的存储中移除。这样的设计有效的减轻了服务端的压力，不然对于更新非常频繁的节点，服务端会不断的向客户端发送事件通知，无论对于网络还是服务端的压力都非常大。

2. 客户端串行执行

   客户端Watcher回调的过程是一个串行同步的过程。

3. 轻量

   3.1. Watcher通知非常简单，只会告诉客户端发生了事件，而不会说明事件的具体内容。

   3.2. 客户端向服务端注册Watcher的时候，并不会把客户端真是的Watcher对象实体传递到服务端，仅仅是在客户端请求中使用boolean类型属性进行了标记。

4. watcher event 异步发送wathcer的通知事件从server发送到client是异步的，这就存在一个问题，不同的客户端和服务器之间通过socket进行通信，由于网络延迟或其他因素导致客户端在不同的时刻监听到事件，由于Zookeeper本身提供了leordering guarantee,即客户端监听事件后，才会感知它所监视znode发生了变化。所以我们使用Zookeeper不能期望能够监控到节点每次的变化。Zookeeper只能保证最终的一致性，而无法保证强一致性。

5. 注册watcher getData、exists、getChildren

6. 触发wathcer create、delete、setData

7. 当一个客户端连接到一个新的服务器上时，watch将会被以任意会话事件触发。当于一个服务器失去连接的时候，是无法接收到watch的。而当clinet重新连接时，如果需要的话，所有先前注册过的watch,都会被重新注册。通常这是完全透明的。只有在一个特殊情况下，watch可能会丢失：对于一个未创建的znode的exist watch,如果在客户端断开连接期间被创建了，并且随后在客户端连接上之前又删除了，这种情况下，这个watch事件可能会被丢失。





#### 客户端注册Watcher实现

1. 调用getData()/getChildren()/exists()三个API，传入Watcher对象
2. 标记请求request,封装Watcher到WatchRegistration
3. 封装成Packet对象，向服务端发送request
4. 收到服务端响应后，将Watcher注册到ZKWatcherManager中进行管理
5. 请求返回，完成注册





#### 服务端处理Watcher实现

1. 服务端接受Watcher并存储

   接受到客户端请求，处理请求判断是否需要注册Watcher,需要的话将数据节点的节点路径和ServerCnxn(ServerCnxn代表一个客户端和服务端的连接，实现了Watcher的process接口，因此可以看成一个Watcher对象)存储在WatcherManager的WatchTable和watch2Paths中去。

2. Watcher触发

   以服务端接收到setData()事务请求触发NodeDataChanged事件为例：

   2.1. 封装WatchedEvent

   