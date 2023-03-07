# 1. 缓存

## 1.1 缓存一致性问题

一段关于好友缓存的代码，伪代码如下：

```c
1.LRUCache friend_list_cache = LRUCache(1000);

// 更新
2.void add_friend(string my_id, string add_id) {
3.	update_friend_list_cache(my_id, add_id);
4.	update_db(my_id, add_id);
5.}

// 读取
6.list<string> get_friend(string my_id) {
7.	if (!cache.contain(my_id)) {
8.		list<string> friend_list = read_db(my_id);
9.		insert_friend_list_cache(my_list, friend_list);
10.		return friend_list;
11.	} else {
12.		return cache.get(my_id);
13.	}
14.}
```

业界常用的方法Cache Aside Pattern：

```c
15.void add_friend(string my_id, string add_id) {
16.	update_db(my_id, add_id);
17.	delete_friend_list_cache(my_id);
18.}
```

总结一下主要是三种：

* 先更缓存，再更DB

  失效情况：更缓存(第3行)->读DB(第8行)->写脏数据到缓存(第9行)。

* 先更DB，再更缓存

  失效情况：线程1更DB->线程2更DB->线程2更缓存->线程1更缓存。

* 先更DB，再让缓存失效

  失效情况：读DB(第8行)->更DB(第16行)->删除缓存(第17行)->写脏数据到缓存(第9行)，这种情况要求读DB再更缓存之间出现了较大的延迟，出现的概率比较低。应该注意更DB再删缓存这2步应该保证原子性。

## 1.2 缓存问题

### 1.2.1 缓存穿透

非法请求一个不存在的key，绕过缓存直接请求mysql，解决办法：

* 在缓存中为这些不存在的key在缓存上设置空值，不适合大量非法key
* 在缓存上设置一层布隆过滤器，如果不在布隆过滤器中，则直接返回

### 1.2.2 缓存击穿

同时大量请求某个合法的key，且不存在缓存中，则大量请求打到mysql，解决版本：

* 用互斥锁限制对mysql的访问，第一个拿到锁的请求把数据加载到缓存后，其他请求直接读缓存。

### 1.2.3 缓存雪崩

redis失效，大量请求到mysql，解决办法：

* 本地缓存+限流降级，redis持久化恢复缓存

### 1.2.4 大量热点key失效

key设置不同的失效时间。

## 2. Redis

### 2.1 常用命令

**事务**

- SETNX key value：如果key不存在，则设置该key为value，并返回1，否则返回0。

**List列表**

* lpush key value：将value插入list(key标识)头部
* rpush key value：将value插入list尾部
* lpop key：从list头部取出一个元素
* rpop key：从list尾部取出一个元素
* lindex key pos：从list取出pos位置(0开始，-1为list尾部)的元素

**无序集合**

* sadd key value：将value添加到集合key中
* scard key：集合key的元素个数

* sscan key cursor COUNT 10：扫描集合，cursor为游标，初始为0，COUNT为每次扫描的数量，返回下次扫描的游标和当前扫描的结果，当返回下次扫描的游标为0时，表示扫描结束。需要注意的是scan系列的命令扫描的结果可能会重复，需要业务过滤掉重复的元素。

**有序集合**
* zadd key score value：将value添加到集合key中，分数为score
* zcard key：集合个数
* zscan key cursor：同sscan。

### 2.2 基于Redis实现分布式锁

在分布式环境下，需要借助第三方工具来实现锁，Redis官方提供了一种基于Redis实现的分布式锁[redlock](<https://redis.io/topics/distlock>)。

**单节点版本**

try_lock返回true，表示成功获取锁，返回false，表示获取锁失败。lock_id为锁的ID，unique_id为每次请求锁的唯一的标识，可以用UUID，lock_valid_time为锁的有效时间，过期后，锁将会自动释放。

release_lock为释放锁的方法，释放锁要带上与single_try_lock相同的lock_id和unique_id。

```c++
bool  try_lock(string lock_id, string unique_id, int leaseTime) {
    // setnx - SET if Not eXists
    return redis.setnx_expire(lock_id, unique_id, leaseTime);
}

void release_lock(string lock_id, string unique_id) {
    if (redis.get(lock_id) == unique_id) { // 避免某个客户端释放另外一个客户端的锁
        redis.del(lock_id);
    }
}
```

这里存在一个问题：某个成功获取锁的客户端A认为自己获得到了锁，此时由于网络延时/CPU抢占/GC，距离请求锁时已经过了很长时间，如果这个时间超过了锁的有效时间，那么锁就可能自动释放，线程B获取锁，此时A还认为自己持有锁，对临界区进行修改。

可以稍微对代码进行修改，解决这个问题，代码如下，try_lock_v2返回的是Lock对象，对临界区进行操作前，需要调用Lock的is_lock_valid判断锁的有效性。

```c++
struct Lock {
    int leaseTime;
    int begin_lock_time;
    bool is_get_lock;
    bool is_lock_valid() {
        int current_time = get_current_timestamp_millisecond();
        return is_get_lock && (current_time - begin_lock_time < lock_valid_time);
    };
};

Lock try_lock_v2(string lock_id, string unique_id, int leaseTime) {
    Lock lock;
    lock.leaseTime = leaseTime;
    lock.begin_lock_time = get_current_timestamp_millisecond()
    lock.is_get_lock = redis.setnx_expire(lock_id, unique_id, leaseTime);
    return lock;
}
```

**集群版本**

Redis集群保证了锁服务的可用性，try_lock返回的是Lock对象，可以通过Lock的is_lock_valid判断是否独占了锁。算法的核心思想是：从超过半数的节点获得锁，就认为获取锁成功。有几点需要注意：

* try_lock是同步调用，为了不阻塞从下一个节点获取锁，可以给同步调用添加一个超时，redlock官方建议是：如果锁的有效时间为10秒，超时为5~50毫秒。
* clock_drift参数是不同机器上的时钟偏差。如果Redis集群中各节点时钟不一致，时钟快的节点上的锁被提前释放，造成两个客户端同时拥有锁。可以设置clock_drift，cluster_try_lock在计算锁有效时间时，会减去系统间的时钟偏差，消除时钟不一致的影响。
* 调用try_lock结束后，如果lock失败，需要调用release_lock释放从集群节点中获得的锁。

```c++
Lock try_lock(string lock_id, string unique_id, int leaseTime, int clock_drift) {
    Lock lock;
    int get_lock_count = 0;
    for (auto server : servers) {
        if (server.try_lock(lock_id, unique_id, leaseTime)) {
            get_lock_count += 1;
        }
    }
    int elapsed_time = get_current_timestamp_millisecond() - lock.begin_lock_time;
    lock.leaseTime = leaseTime - elapsed_time - clock_drift;
    lock.lock_begin_time = get_current_timestamp_millisecond()
    if (lock.leaseTime > 0 && get_lock_count > servers.size()/2) {
        lock.is_get_lock = true;
    } else {
        lock.is_get_lock = false;
    }
    return lock;
}

void release_lock(string lock_id, string unique_id) {
    for (auto server : servers) {
        server.release_lock(lock_id, unique_id);
    }
}
```

try_lock如何调用到指定server？
一种做法是在lock_id上制定hash tag。从master查到slot信息，比如每个master管理什么slot。
指定key的副本数replica，比如为3。根据slot反查出一个key，将key作为hash tag放到lock_id上。这样这个lock_id就可以指定调用到某个master。

### 2.3 redis实现信号量

```java
public class Semaphore_Lock {
    public static String acquire_semaphore(Jedis redis,String sem_name,int limit,long timeout){
        String identifier=UUID.randomUUID().toString();
        long now=System.currentTimeMillis();
        Pipeline pipeline=redis.pipelined();
        //清理其他持有者过期信号量
        pipeline.zremrangeByScore(sem_name, 0, now-timeout);
        pipeline.zadd(sem_name,now,identifier);
        Response<Long> rank=pipeline.zrank(sem_name, identifier);
        pipeline.syncAndReturnAll();
        if((Long)rank.get()<limit){
            System.out.println(Thread.currentThread().getName()+"  identifier rank :"+rank.get());
            return identifier;
        }else{
            System.out.println(Thread.currentThread().getName()+"  identifier rank :"+rank.get()+",too late");
        }
        redis.zrem(sem_name, identifier);
        return null;
    }
    public static Long release_semaphore(Jedis redis,String sem_name,String identifier){
        return redis.zrem(sem_name, identifier);
    }
}
```

# 3. Redis Cluster

## 3.1结构

Redis Cluster保证了可用性，不保证强一致性。Redis Cluster为N主N从，主从复制的结构。从节点不保证数据和主节点一致，如果有一致性的需求，可以禁用掉从读。

## 3.2 Sharding

**Hash slot**

Redis Cluster将key空间分为16384个槽位(slot)，又称为哈希槽(hash slot)，每个master节点处理一个hash slot子集，hash slot的计算方式如下：

```python
def HASH_SLOT(key):
    return CRC16(key) % 16384
```

如果希望两个不同的key映射到同一个slot，可以用hash tag，带hash tag的slot计算方式如下：

```python
def HASH_SLOT(key):
    if "{" in key and "}" in key:
        s = key.index("{") + 1
        e = key.index("}")
        if e > s:
            return CRC16(key[s:e]) % 16384
    return CRC16(key) % 16384
```

**Hash slot VS Consistant hashing**

一致性哈希在变更节点时，需要遍历节点的所有key，判断key是否需要迁移。Hash slot在变更节点时，只需要遍历slot，判断slot是否迁移。

## 3.3 数据迁移

当cluster新增master或者下线master时，涉及到slot的重新分配，即从一个master迁移到另一个master。如果需要将slot X从master A迁移到master B，需要做如下操作：

```shell
master B执行：CLUSTER SETSLOT X IMPORTING A # 设置B slot X为从A迁入状态
master A执行：CLUSTER SETSLOT X MIGRATING B # 设置A slot X为迁出到B状态
master A执行：MIGRATE HOST_B PORT_B "" 0 5000 KEYS key1 key3 # 迁移数据，可以选择迁移单个key或者多个key。键迁移期间，A和B处于阻塞状态，保证一致性。
master B执行：CLUSTER SETSLOT X NODE B # 设置X为B的slot
```

下图为迁移过程客户端访问slot X的结果，基于hash slot的寻址方式需要客户端记录slot和master节点的映射关系。如果某个slot不在当前访问的master节点，就会返回MOVE，客户端缓存slot和master节点的信息。迁移时，旧key仍需要在源节点处理，新key需要到新节点处理，所以需要增加ASK状态，表示仅在下次访问新key时，转到新节点，并不更新客户端缓存，如果缓存被更新，旧key也被路由到新节点了，显然是错的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/de70bf48906576e0497431514d1f1114.png#pic_center)

## 3.4 Gossip协议

Redis Cluster使用了gossip协议。gossip协议中，每个节点A执行如下步骤：

```shell
Step1：选择K个相邻节点，对每个选择的节点B，A与B其交换信息
Step2：等待T秒
Step3：执行Step1
```

上述算法中有两处决定了gossip协议的性能：

* 如何选择相邻的K个节点：一种策略是随机选择，下个周期会从剩余节点中随机选择。
* 如何交换信息：交换信息分三种方式：
  * push：A将其{key, value, version}发送给B，B更新A中比自己版本高的数据。
  * pull：A将其{key, version}发送给B，B把比A版本高的数据{key, value, version}发给A。
  * push/pull：A将其{key, value, version}发送给B，B更新A中比自己版本高的数据，再把比A版本高的数据{key, value, version}发给A。

gossip协议最大的优点是去中心，消息可靠投递。缺点是消息传递到所有节点需要 `O(logN)* T` 秒，N是节点数量，且消息有冗余。




