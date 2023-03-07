

# 游戏服务引擎

游戏服务端区别于互联网服务端，互联网后台多是由多个微服务组成，如登录，商品，订单，支付服务等，后台数据持久化存储在DB中，并利用缓存提高数据访问速度。

## ECS(entity component system)

游戏服务端通常使用一种称为ECS的架构，每个entity代表一个对象，可以是玩家，NPC，也可以是服务。

为了支持entity之间进行RPC通信，entity有MailBox，由IP端口和EntityID确定，集群间的Entity通过MailBox进行RPC调用。

客户端存在一个和服务端对应的entity，称之为avatar，代表玩家。客户端和服务端通过KCP(可靠UDP)协议传输数据。

**AvatarEntity**

代表玩家，是所有玩家控制的Entity的父类

* 持有一个ClientProxy的对象，通过该对象和客户端的entity通信，ClientProxy封装了对客户端entity对象的调用。

* 每个客户端在服务端都只有一个ClientProxy和它通信，所以同一个时间同一个客户端只能控制服务端的一个AvatarEntity，但客户端需要控制服务端不同的entity的时候，就需要服务端的entity通过give_client_to来转移ClientProxy的所有权。



## 单服结构

下图为传统的单服结构，这里的单服并非单机，而是对应一组Gate，Game，GameManager组成的进程，每个进程都是单epoll线程。

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/c17486b9d2a0ca16b17b5e9982623ced.png#pic_center)


* Gate负责和客户端连接，转发客户端消息到Game进程上，还可以做单服内消息的广播(广播消息到Game进程)，Gate与所有的Game，GameManager建立连接，Gate之间不连接。
* Game是承载游戏逻辑的进程，可以理解成运行一个Spring容器，开发者可以通过开发Entity来实现逻辑。
* GameManager负责整个单个集群的元数据管理，如管理Game，Gate的地址，创建全局的Entity(一般是服务)等。多个GameManager是避免单点故障。

**服务发现**

Game服务器通过GameManager发布自己的地址信息：Game服务启动之后都会注册自己到GameManager中，同时也会发送keep-alive给GameManager，因此Gate服务器可以通过GameManager得到所有Game服务器的地址。



**客户端登陆**

* 客户端会有每个游戏服的Gate服务器的列表（Gate服务器列表可以通过更新本地配置文件，也可以每次启动时候连接一个bootstrap服务器获得）
* 客户端按照列表逐个尝试Gate服务器，直到找到一个可以连上的。
* Gate服务器随机选择一个Game服务器，转发认证请求
  * Gate服务器记录下该客户端和game服务器的映射关系，以后所有来自该客户端的请求都转发给该game服务器，直到映射关系变化
  * 认证请求用非对称加密，game服务器需要有私钥
* Game服务器负责向URS进行用户认证
  * 认证成功生成会话密钥
  * 告之gate服务器和客户端会话密钥session_key
  * 以后的游戏通信都通过会话密钥
* 客户端使用会话密钥加密消息向Gate服务器请求开始游戏
  * Gate服务器会解密和解压缩客户端消息，把解密的消息转发给Game服务器
  * Game服务器发给客户端的消息，会被Gate服务器进行加密和压缩
  * 因此Game服务器无需处理加密和压缩解压缩



**游戏开始流程**

* 客户端第一次连接的时候，Gate服务器会选择一个Game服务器

* 该Game服务器会负责创建对应的Entity(Account)来进行认证

* 之后游戏逻辑需要自己根据用户Account的信息判断Avatar应该所在的场景以及Game服务器，并进行必要的entity迁移（从一个Game服务器迁移到另外一个Game服务器）

* 当Entity迁移的时候需要通知Gate服务器客户端的映射关系变化，Gate服务器会记住每个客户端当前所在的Game服务器，一个客户端同时只能在一个Game服务器上



**Entity迁移**

* Entity的迁移过程由Entity当前所在Game服务器发起

* 老的Game服务器通知gate服务器缓存该entity的所有消息直到新的Game服务器ready

* 老的Game服务器得到gate服务器确认的rpc之后，通过Game Manager中转将entity迁移到新的game 服务器

* 迁移成功后，新的服务器过Game Manager通知老的Game服务器，由老的Game服务器通知Gate服务器该Entity的新地址
* Gate服务器把缓存的消息发给新的Game服务器，并开始正常流程。
* 老的game服务器过一段时间之后会清理掉该entity

让gate缓存转发消息，而不是让老的game服务器转发消息主要是希望保证消息的有序性。因为如果让老的game服务器转发消息，可能会出现gate服务器和老的game服务器同时在给新的game服务器发送客户端消息，容易出现消息乱序。



**服务的实现**

服务也是通过Entity实现的，可以指定某个服务有多少个shard(通过配置)，调用时制定调用的方式：

* call_shard_service：同一个shard key可以调用到相同的服务Entity
* call_rand_service：无状态的调用



## 多服结构

单服结构的缺点是所有游戏逻辑(登录，社交，排行，工会，匹配，战斗等)都在一个服上实现，比如5V5 moba游戏，一个进程能承载的最多战斗为4场，要想实现10万人同时战斗，那这个服至少要有2500个进程，会导致GameManager负载过高，单点成为瓶颈。

于是出现了多服结构，如下图所示是一个大厅服，一个中心服，N个战斗服的架构，每个服都是一个单服结构，每个服都有Hostnum作为唯一的ID。

每个服的启动配置中指定了hostnum和服的类型，大厅服和战斗服配置中还指定了中心服的hostnum，这样可以调用到中心服的服务。
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/c879d9355413a27187c61f2c8bc2d23b.png#pic_center)



**大厅服**

所有玩家都在一个大厅服上登录，在玩家看来，都是在一个服进行登陆。因为大厅服只负责战斗外的一些逻辑，不会无上限的扩展Game进程，所以单服可以承载大量的玩家，也可以扩展大厅服的数量。



**中心服** 

多服结构中只存在一个中心服，负责全局的玩法，如跨服匹配等，也负责管理战斗服的资源，负载均衡等。



**战斗服**

只负责战斗逻辑，无状态，不依赖存储，可以动态的增删战斗服。战斗服的状态(多少个game进程，每个进程目前有多少个战斗)需要定时向中心服上报，这样中心服就可以全局感知有多少个战斗服，每个战斗服的负载(能安排多少个战斗，以及目前有多少个战斗正在进行)。

当玩家匹配完成后请求分配战斗资源：

* 中心服依据负载均衡策略，比如选择最小负载的战斗服，调用这个服创建战场(Space)。
* 战斗服收到请求创建Space的消息后，选择负载最小的game创建并初始化战场Space，Space里面会包含所有玩家(BattleAvatar)和场景等元素。
* 战斗服将Space创建成功的信息返回给中心服
* 中心服将战场信息，包括战斗服gate的地址，战场的mailbox告诉客户端
* 客户端重新登录(根据gate)到战场所在战斗服，再把Account迁移到战场所在进程
* 迁移成功后，找到这个进程的Space对象，再找到自己的BattleAvatar，把这个BattleAvatar控制权交给客户端。



**HUB服** 

实现跨服调用的一组服务，只做消息转发。entity的mailbox新增了一个hostnum，hub服会根据hostnum将消息投递到目标entity所在的服。


**多服登录流程**
![在这里插入图片描述](https://img-blog.csdnimg.cn/ac1a28ef597a4b81a31c84b1c3dc332a.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW5waW5nemp1,size_16,color_FFFFFF,t_70#pic_center)
