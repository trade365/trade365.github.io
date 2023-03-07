# 1. 网络

滑动窗口协议：https://juejin.im/post/5c9f1dd651882567b4339bce

拥塞控制：https://mp.weixin.qq.com/s?__biz=Mzg2NzA4MTkxNQ==&mid=2247486586&idx=2&sn=88e9835deb2c1b85ea42b5de13b81e72&scene=0#wechat_redirect

TCP四次挥手：

* 主动关闭方FIN
* 被动关闭方ACK
* 被动关闭方进入close wait
* 被动关闭方FIN
* 主动关闭方ACK，同时进入time wait状态。

CLOSE_WAIT过多原因：被动关闭方没有主动关闭连接。

**epoll**

epoll一共两种模式，水平触发(LT)和边缘触发(ET)，主要的区别在于对读，写数据的处理:

| 操作 | 读                                              | 写                                                           |
| ---- | ----------------------------------------------- | ------------------------------------------------------------ |
| LT   | 只要读缓冲区有数据可读，就返回EPOLLIN           | 只要写缓冲区有空间可写，就返回EPOLLOUT                       |
| ET   | 读缓冲区从无数据变成有数据可读，返回一次EPOLLIN | 写缓冲区从不可写变成可写(不可写通常是写缓冲区满了)，返回一次EPOLLOUT |

LT模式下，如果注册了EPOLLOUT事件，epoll_wait几乎都会返回EPOLLOUT，除非写缓冲区满了，所以LT模式下EPOLLOUT事件需要特殊处理。下面给出LT模式和ET模式下处理的读写的伪代码:

```java
//LT初始化
epoll_ctrl(connect_socket, EPOLLIN, ADD);

// LT读
void lt_handle_read() {
    // 如果读不完，下次epoll_wait一定会返回EPOLLIN
    while((n = read(connect_socket, read_buffer)) > 0) {
        // read_buffer为当前循环读取到的数据
    }
    // n=0表示对端关闭连接，需要关闭socket
    if (n < 0 && errno == EAGAIN) {
        // Resource temporarily unavailable, 无数据可读
    }
}

// LT写
void lt_handle_write() {
    // 一次性发满写缓冲区，这里写满的意图是：判断是否需要添加EPOLLOUT
    while((n = write(connect_socket, write_buffer)) > 0) {
        // write_buffer为当前循环发送的数据
    }
    if (write_left == 0) { // 数据发送完毕，删除EPOLLOUT
        epoll_ctrl(connect_socket, EPOLLOUT, DELETE);
    }
    if (n < 0 && errno == EAGAIN) {
        // Resource temporarily unavailable, 写缓冲区满了，添加EPOLLOUT
        epoll_ctrl(connect_socket, EPOLLOUT, ADD);
    }
}
```

```java
//ET初始化
epoll_ctrl(connect_socket, EPOLLET | EPOLLIN | EPOLLOUT, ADD);

// ET读
void et_handle_read() {
    // 这里要把数据读完，否则就是BUG了
    while((n = read(connect_socket, read_buffer)) > 0) {
        // read_buffer为当前循环读取到的数据
    }
    if (n < 0 && errno == EAGAIN) {
        // Resource temporarily unavailable, 无数据可读
    }
}

// ET写
void et_handle_write() {
    // 一次性发满写缓冲区
    while((n = write(connect_socket, write_buffer)) > 0) {
        // write_buffer为当前循环发送的数据
    }
    if (n < 0 && errno == EAGAIN) {
        // Resource temporarily unavailable, 写缓冲区满了，等待可写触发EPOLLOUT
    }
}
```

于是总结出两种模式的优缺点:

| 操作 | 优点                                                         | 缺点                                                         | 应用场景                              |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------- |
| LT   | 编码灵活，不容易出错                                         | 多两次epoll_ctrl调用                                         | 写数据小，不会触发EPOLLOUT，比如Redis |
| ET   | 比LT模式少了动态添加和删除EPOLLOUT事件，少了2次epoll_ctrl调用 | ET模式只在状态发送变化时触发一次，在处理读时，需要一次性把读缓冲区读完，在处理写时，需要一次性把缓冲区写满 | 写数据大，频繁触发EPOLLOUT，比如Nginx |

# 2. RPC

## 2.1 dubbo

Provider xml配置(可以配置多个服务)：

```xml
<dubbo:service interface="com.xxx.DemoService" ref="demoServiceImpl"/>
<bean id="demoServiceImpl" class="com.xxx.impl.DemoServiceImpl" />
```

Consumer xml配置：

```xml
<dubbo:reference interface="com.xxx.DemoService" id="demoServiceImpl"/>
```

dubbo优先使用consumer端的配置，如果consumer端配置缺失，使用provider端配置。

### 2.1.1 分组

dubbo有接口有多种实现时，可以使用dubbo的分组(group功能)

### 2.1.2 泛化调用

不需要引入接口定义就可以进行远程调用

### 2.1.3 线程模型

https://www.cnblogs.com/xhj123/p/9095278.html

dubbo服务端有一个线程池，可以配置派发方法在线程池上执行或者在IO线程上执行(dispatcher配置)。

### 2.1.4 服务导出

dubbo可以自定义ExportListener，在服务导出前，Spring初始化Bean之后做一些自定义的逻辑，比如调用其他dubbo服务。

服务导出流程：https://dubbo.apache.org/zh-cn/docs/source_code_guide/export-service.html，总结一下分如下几步：

* 每个服务都是一个ServiceBean，监听Spring的ContextRefresh事件(默认最低的优先级)

* 检查配置，根据配置装配URL(protocol)
* 将每种协议(默认为dubbo)，导出到registryUrls(**doExportUrlsFor1Protocol**)。
* 生成Invoker(包装了wrapper，doInvoke即调用wrapper的invokeMethod，根据方法名调用服务对应的实现)。
* 导出服务：本地和远程分别导一次，远程是调用RegistryProtocol的export(Invoker方法)，里面会调用DubboProtocol的export方法，关键代码如下：

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker); //启动netty
    final Registry registry = getRegistry(originInvoker); // 创建zkClient
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    registry.register(registedProviderUrl); // 创建provider ZK临时节点
    ...
}
```

### 2.1.5 调用拦截

filter可以对调用进行拦截，通过filter链实现filter功能，先处理filter，再调用invoker：https://dubbo.apache.org/zh-cn/blog/first-dubbo-filter.html

指定filter顺序：https://www.cnblogs.com/mumuxinfei/p/9231310.html

### 2.1.6 调用流程

Dubbo使用Javassist生成接口的动态代理类，客户端和服务端都是通过Invoker对象的doInvoke方法来实现远程调用流程。

客户端的doInvoke方法将RPC接口名，方法名和参数序列化成二进制字节，发送到网络。具体流程：

* 调用动态代理对象的RPC方法
* 调用InvocationHandler的invoke方法
* 调用Invoker的doInvoke方法

服务端的Invoker是ProxyFactory的getInvoker方法生成，doInvoke方法调用wrapper的invokeMethod方法，wrapper的invokeMethod方法会根据方法名调用接口实现类的方法。如下代码所示：

```java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
	// 为目标类创建 Wrapper
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    // 创建匿名 Invoker 类对象，并实现 doInvoke 方法。
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
			// 调用 Wrapper 的 invokeMethod 方法，invokeMethod 最终会调用目标方法
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

proxy是接口的实现，Wrapper的代码是运行时动态生成编译的，代码大致如下：

```java
public Object invokeMethod(Object object, String string, Class[] arrclass, Object[] arrobject) {
    DemoService demoService = (DemoService)object;
    // 根据方法名调用指定的方法
    if ("sayHello".equals(string) && arrclass.length == 1) {
        return demoService.sayHello((String)arrobject[0]);
    }
}
```

## 2.2. GRPC

GRPC是protobuf风格的RPC，步骤如下：

* 定义rpc方法的proto文件
* 生成服务端和客户端的代码
* 服务端引用生成的代码，实现RPC方法
* 客户端引用生成的代码，实现RPC的远程

客户端调用流程：

```c++
// 客户端
void DoSearch() {
  channel = new MyRpcChannel("somehost.example.com:1234"); // channel封装网络连接，数据收发
  controller = new MyRpcController;
  service = new SearchService::Stub(channel); // 获取客户端的Stub
  request.set_query("protocol buffers");
  // 用客户端的Stub进行远程调用
  // 每个方法会调用channel的CallMethod
  // channel的实现类应该将方法名，参数等打包，发送到网络
  service->Search(controller, request, response, protobuf::NewCallback(&Done));
}
```

服务端调用流程：

```c++
//服务端
class ExampleSearchService : public SearchService {
 public:
  void Search(protobuf::RpcController* controller,
              const SearchRequest* request,
              SearchResponse* response,
              protobuf::Closure* done) {
    // 处理请求，返回结果
    done->Run(); //执行客户端的回调
  }
};

int main() {
  MyRpcServer server; // TCP或其他服务器
  protobuf::Service* service = new ExampleSearchService; // 创建service实例
  server.ExportOnPort(1234, service); // 将网络和服务绑定起来
  server.Run(); // 运行服务器
}
// 服务器网络层收到数据后，将数据解包，解析出方法index(减少数据传输)和调用参数
// 调用service的CallMethod方法(CallMethod方法是由protoc生成的代码)
// SearchService的CallMethod方法会根据RPC方法的index调用子类的RPC实现
```

### 2.2.1 grpc name resolver

grpc java的channel通过NettyChannelBuilder构建，需要通过NameResolverProvider来指定NameResolver，需要自己来实现NameResolver，将服务的ip:port列表传给grpc。

### 2.2.2 grpc load balancer

grpc默认使用RoundRobinLoadBalancer，也可以定义自己的LoadBalancer，在loadBanlaner中，重要的方法是pickSubchannel和handleResolvedAddresses

* handleResolvedAddresses：将resolve的ip:port构建成grpc channel
* pickSubchannel：根据LB策略，选择一个channel

# 3. 限流

令牌桶算法

有一个生产线程向令牌桶中以r的速率(通常是QPS)存放token，桶的容量是r。如果超出了桶的容量则丢弃token。消费线程请求n个token，可以请求成功或者阻塞一定时间，请求成功则从桶中删除n个token。

RateLimter

RateLimter保证N秒内，平均每秒分发r个token，但是不能保证每秒的token都在r内。本秒内超发的token将由下一个消费线程偿还等待时间。

```java
RateLimiter limiter = RateLimiter.create(5000); // r=5000
while (true) {
    System.out.println(limiter.acquire(3000));
    System.out.println(limiter.acquire(2000));
}
```

```java
RateLimiter limiter = RateLimiter.create(5000);
while (true) {
    System.out.println(limiter.acquire(10000)); // 超发，不会等待2秒，而是等待0.4秒
    System.out.println(limiter.acquire(2000));
}
```

# 4. 服务发现

## 4.1. etcd

### 4.1.1 服务发现

etcd做服务发现的思路如下：

* Service进程将服务名，编号(同名的服务可能会有多个实例，以编号区分)和地址以kv的形式提交到etcd，并设置租期lease。超过了租期，且没有续租，etcd会将key删除。所以Service进程要定期调用lease的刷新方法，即续租。
* 如果想获取服务信息，可以通过etcd的get方法获取服务，或者通过watch方法监听服务变化。watch的含义是：如果对key(对应watch)或者key前缀(对应watch_prefix)进行修改，就会返回该变化，进而得知服务的变化，如新增服务或者下线服务。

服务发现有几点需要注意的：

* provider服务需要先启动，保证能对外提供服务后，再注册到ETCD
* provider服务计划关闭后，需要先从ETCD注销，经过一段优雅停机后，再关闭服务。
* consumer需要有熔断provider的策略。

### 4.1.2 Raft协议

etcd实现了Raft协议，Raft协议分选主(Leader Election)和日志复制(Log Replication)两部分。

### 4.1.3 选主

Raft为集群中的节点定义了三种状态，分别是Follower，Candidate，Leader。

选主步骤如下：

* Candidate状态的节点会向其他节点发送vote_request，参数需要带上当前节点的term，term表示第几轮选举，每当Candidate发起新一轮的投票请求，term就会加1，带有过期term的消息不会被处理(如heartbeat，vote_request，vote)。
* Follower节点收到vote_request，如果Follower节点检测Leader故障(**election timeout**时间内没收到heartbeat)，且在该term并未投过票，则投票给Candidate。
* 某Candidate收到超过半数的投票，成为Leader。
* Leader节点每隔**heartbeat timeout**时间就会给其他节点发送心跳。**election timeout**的时间是150~300毫秒之间的随机值。

### 4.1.4. 日志复制

Raft协议规定只能通过Leader来写，可以从任意节点读。Leader通过日志复制的方式将数据同步到其他节点。Leader需要更新的数据封装在Append Entries消息中，通过heartbeat传输给其他节点。写数据一共分两个阶段：

* 第一阶段：写日志，不提交。Append Entries={在日志中的编号，要更新的数据，不提交}。当Follower收到Append Entries消息后，如果日志不冲突，将Append Entries插入到日志中，回应Leader可写。当Leader收到超过半数的节点回应时，发起第二阶段写。
* 第二阶段：提交。Append Entries={在日志中的编号，提交}，Leader回应客户端写成功。当Follower收到Append Entries消息后，在日志中提交数据。


https://ms2008.github.io/2019/12/04/etcd-rumor/

### 4.1.5 线性一致性

**Replicated State Machine**，**Read Index**，**Lease Read**，**Follower Read**

https://blog.csdn.net/z69183787/article/details/112168120



## 4.2 Zookeeper

### 4.2.1. Curator

CuratorFramework封装了ZK原始API

```java
// 连接ZK
CuratorFramework client = CuratorFrameworkFactory.builder()
        .connectString("localhost:2181")
        .connectionTimeoutMs(15000)
        .sessionTimeoutMs(40000)
        .retryPolicy(new RetryNTimes(5, 1000))
        .namespace("test_ns")
        .build();
client.start();
```

ZK和ETCD不同的地方是ZK按照目录的方式组织数据，而ETCD更像是KV的方式，比如：

```java
client.create().withMode(CreateMode.PERSISTENT).forPath("/par", "hello".getBytes());// 直接创建/par/child路径会出错
client.create().withMode(CreateMode.PERSISTENT).forPath("/par/child", "hello".getBytes());
```

而在ETCD中，是可以绕过前缀直接put一个完整的路径。

创建ZK节点(znode)时，可以指定节点存活方式：

* PERSISTENT：创建持久化节点
* PERSISTENT_SEQUENTIAL：持久化节点，节点名后带上一个自增的ID
* EPHEMERAL：创建临时节点，客户端断连后，节点会被删除
* EPHEMERAL_SEQUENTIAL：创建临时节点，节点名后带上一个自增的ID

遍历某个路径，查询，并删除子节点：

```java
List<String> childs = client.getChildren().forPath("/par");for(String s : childs) {  // 查询 
  System.out.println(new String(client.getData().forPath("/par/"+s)));  // 删除，如果s目录不为空，则抛异常
  client.delete().forPath("/par/"+s);
}
```

改数据：

```java
client.setData().forPath("/par", "world".getBytes());
```

watch模式：

```java
PathChildrenCache cache = new PathChildrenCache(client, "/par", true);
cache.getListenable().addListener((client, event) -> {
  // CuratorFramework client, PathChildrenCacheEvent event
});
cache.start();
```

事务：

```java
client.inTransaction()
  .create().withMode(CreateMode.PERSISTENT).forPath("/par3", "hello".getBytes())
  .and()        
  .create().withMode(CreateMode.PERSISTENT).forPath("/par3/child")
  .and().commit();
```

选主：

```java
LeaderLatch leaderLatch = new LeaderLatch(client, "/latchpath", "test_id", LeaderLatch.CloseMode.NOTIFY_LEADER);
leaderLatch.addListener(new LeaderLatchListener() {
    @Override
    public void isLeader() {
        System.out.println("is leader");
    }

    @Override
    public void notLeader() {

    }
});
leaderLatch.start();
```

监听和ZK的连接状态变化：

```java
client.getConnectionStateListenable().addListener(new ConnectionStateListener() {
    @Override
    public void stateChanged(CuratorFramework client, ConnectionState newState) {
    }
});
```

Curator客户端的状态：
* CONNECTED：第一次连接
* SUSPENDED：对应原生的Disconnected，连接断开时触发
* LOST：对应原生的Expired，session过期时触发
* RECONNECTED：包括sessionid不变的重连和sessionid变化的重连，如果客户端建立了EPHEMERAL节点,必须在此事件中判断sessionId。对应sessionId不变的情况，连接断开期间watch的事件不会丢失，如果sessionId变化，则期间watch的事件会丢失。

当客户都和ZK集群断开连接时，客户端会尝试重连，如果在session timeout时间内连接成功，则客户端收不到stateChanged的回调，如果在session timeout之内没有重连成功，ZK集群会认为该客户端的会话已经结束，并清除和这个session有关的数据(临时节点等)，需要注意的是：清除时间和客户端收到LOST或者SUSPENDED的时间无法确定先后。

ZK不适合作为注册中心的观点：https://www.jianshu.com/p/d9fc146e7e9a


# 5. 序列化
## 5.1 protobuf
proto文件，用protoc生成java或者CPP的文件，版本要一致，不然可能序列化的数据不对。

# 6.HTTP
## 6.1 浏览器缓存
https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control
默认的Cache Control是private
