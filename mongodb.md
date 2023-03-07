# MongoDB

MongoDB属于NoSQL。在MongoDB中，一个db(对应MySQL的db)包含若干collection(对应MySQL的table)，一个collection存储若干json文档(对于MySQL的行)，MongoDB不适合处理事务。

## Index

索引主要分两种类型：ranged index和hashed index。Hashed index用在hashed sharding，hashed index只能用于等值查找，不能用于范围查找。

ranged index支持单个字段和复合字段，1表示升序，-1表示降序：

* 单个字段：升序和降序都可以，因为MongoDB既可以正向遍历也可以反向遍历。
* 复合字段：以索引{userid: 1, score: -1}为例，因为只能沿一个方向遍历，只支持如下两种排序查找。

```sql
db.collection.find().sort({userid: 1, score: -1})
db.collection.find().sort({userid: -1, score: 1})
```

### 磁盘索引-B树/B+树

MongoDB使用B树索引，MySQL使用B+树索引。

#### B树

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/47af20cfafb34a0a96775356512eced6.png#pic_center)


&ensp;一个m阶B树的特点如下：

* 每个节点的key都存储了指针指向key对应的value。
* 每个节点存储的key都不相同。
* 每个节点有m个指针槽位，指向叶子节点，有m-1个key槽位。
* 如果根节点非叶子节点，则有[2, m]个子节点。
* 非根节点有[m/2, m]个子节点。

#### B+树

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/466f0c5d9485863f4700e11a156053e6.png#pic_center)


一个m阶B树的特点如下：

- 每个节点有m个指针槽位，指向叶子节点，有m个key槽位。
- 如果根节点非叶子节点，则有[2, m]个子节点。
- 非根节点有[m/2, m]个子节点。
- 叶子节点存储了指针指向相邻的叶子节点。

可见B+树相对于B树改动的内容如下：

* B+树非叶子节点并不存储value的指针，可以让出更多空间给key，进而降低树的高度，降低磁盘IO数。但是每次查找都需要访问到叶子节点，查找效率没有B树高。
* B+树叶子节点存储了指向相邻叶子节点的指针，方便范围查找。

## Sharding

MongoDB是分shard的，客户端只需要指定shard key，其他路由等操作对客户端是透明的。

mongos作为独立的进程，安装在每台应用服务器上，应用中的MongoDB客户端(如pymongo)只需要连接本机的mongos，mongos会将客户端的请求路由到Shard上进行处理。

每个Shard由若干chunk组成，chunk有一个shard key值的上限和下限，shard key值落在其中，则属于这个chunk。chunk增长到一定容量就会分裂，并且可以通过balancer在Shard间迁移。

Config Server负责Shard上chunk的管理等，如运行balancer迁移chunk。

每个shard和Config Server都是一个replica set，每个replica set由一个Primary和若干个Secondary组成，Primary负责读写，Secondary只能读，如果Primary和Secondary加起来是奇数个，则可以添加一个Arbter，只用作投票，并不复制数据，如果Primary Fail，会投票选出一个Secondary作为Primary。

Shard key决定了数据位于哪个chunk，进而决定了chunk的数量，以及chunk上collection的数量，理想的shard key可以让数据均匀的分布在chunk上。用于shard key的字段必须建索引，并且值是不可以改变的。MongoDB一共有两种sharding方式：hashed sharding和ranged sharding。

### Hashed sharding

hashed sharding使用某个字段作为哈希索引，会计算这个字段的哈希值，并取该值作为shard key值。用作hashed sharding的shard key应该具备如下特点：

* 不同的值很多。举个反例就是bool值，正例是电话号码。
* 可以是单调增长的，如时间戳，或ObjectID。

### Ranged sharding

ranged sharding使用数值型/字符串的字段作为shard key，数值型的字段取其值作为shard key值，字符串型的字段取字典值作为shard key值。所以相邻的shard key会分布到相近的chunk，方便范围查找。用作ranged sharding的shard key应该具备如下特点：

- 不同的值很多。
- 值的频率低，即相同的值出现的次数低。
- 非单调增长：如果是单调增长的，写操作会集中在maxKey所在的chunk。

### Compound shard key

时间戳是单调增长的，不适合单独用时间戳作ranged sharding的shard key，因为写操作会集中在一个chunk。但常见的操作就是查找最近的文档，用时间戳作ranged sharding的shard key会提高读操作的性能。举例如下，查找用户Bob最近的10封邮件：

```sql
db.mail.find({userid: 'Bob'}).sort({"email_date":-1}).limit(10)
```

一种方法是使用compound shard key，用法为{"userid": 1, "email_date": 1}，同一个userid相邻的email_date会分布在相邻的chunk，方便查找，userid会将写压力分摊到各chunk。但对于同一个usedid，写操作会集中在某个chunk。

### Tips

* 如果分了shard，update_one(还有delete_one/find_one_and_update)的查询条件需要**包含**shard key，不然会报错，因为mongo需要确定到哪个shard去执行update_one。如果查询条件不能包含shard key，可以用update_all。

