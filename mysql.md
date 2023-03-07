# 1. 隔离级别

数据库隔离级别，隔离级别从高到低：

* Serializable(串行化)：事务串行执行。
* Repeatable-Read(可重复读)：一个事务里对某个数据的查询结果都是一样的，不管其他事务是否修改了该数据。也就是解决了幻读和不可重复读的问题。
* Read Committed(读已提交)：一个事务不会读到其他事务未提交的修改结果，也就是说如果其他事务提交，则就会读到提交的结果。
  * 如果其他事务插入了数据，本事务读到了插入的数据，则为幻读；
  * 如果其他事务修改了数据并提交，本事务读到了提交前后的数据，则为不可重复读。
* Read Uncommitted(读未提交)：一个事务会读到其他事务未提交的修改结果，如果其他事务回滚，则会读到回滚前的结果。

# 2. SQL

* unique：每个表可以有多个unique约束，但是只能有一个primary key约束。被unique和primary key的字段都会作为mysql索引
* union: https://www.w3school.com.cn/sql/sql_union.asp
* in和exists的区别：[in适合从表小，exists适合主表小](https://blog.csdn.net/weixin_41979002/article/details/118730611)

# 3. 分页

limit offset, N：扫描offset+N条数据，并把从offset开始的N条数据返回。连续分页查询的同时如果修改数据库，可能会产生查询数据丢失的情况，如下。可以通过增加修改时间戳来解决。

|                       |
| --------------------- |
| limit 0, 2(展示前2条) |
| 删除前2条数据的某一个 |
| limit 2, 2(数据丢失)  |

## 3.1  超大分页优化

log_tbl表特别大，service是非主键索引：

* **Bad case**: select * from log_tbl where service="xxx" limit 1000000, 5，这里需要回表1000005数据，返回5条，查询很慢。
* **Good case**: select * from log_tbl as a join (select id from log_tbl where service="xxx" limit 1000000, 5) as b where a.id = b.id; 这里只需要回表5条数据，大大缩短耗时。



# 4. 索引类型

## 4.1  聚簇索引和非聚簇索引

* 聚簇索引：索引数据和表数据放在一起存储。B+树的叶子节点存储数据，辅助索引叶子节点存放的是主键，需要回一次表，才能查到数据，好处是：当数据页分裂时，辅助索引不需要变化，而非聚簇索引都要变。
* 非聚簇所以：索引和表数据分离。B+数的叶子节点存储数据的指针(索引)。

如下图所示，InnoDB只有一个聚簇索引(主键)，其余均为非聚簇索引(辅助键)。MyISAM均为非聚簇索引。

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/77b15929d4fadb248e6fbc1755dcd20b.png#pic_center)



## 4.2  回表查询和索引覆盖

为了避免回表查询，可以用索引覆盖的方式，将需要查询的字段包含到索引上，

```java
create table test_user (
	id bigint(20) unsigned not null auto_increment,
    uid bigint(20) unsigned not null,
    nickname varchar(20) not null,
    primary key(id),
    key(uid, nickname)
);

explain select nickname, uid from test_user where uid = 2;
```

联合索引遵守最左原则，mysql会根据最左边的字段建立B+数索引，然后依次按照剩余字段进行排序。索引如下：

1,2,3|1,2,4|1,3,3|1,3,4

## 4.3  Mysql查询优化

至少是range级别：https://www.modb.pro/db/25959


# 5. 锁

## 5.1 按照锁的粒度

* 行级锁：针对行加锁，粒度小，开销大，并发度高。
* 表级锁：针对表加锁，并发度低，不容易死锁。
  * lock table user read;
  * lock table user write;

**行锁是通过索引来实现的**。如果要想使用行锁，查询条件必须带上索引，否则使用的是表锁，并发度降低。

## 5.2 按照锁级别

* 共享锁(读锁)：和读写锁的读锁类似

  用法：select ... lock in share mode;

* 排他锁(写锁)：和读写锁的写锁类似

  用法：select ... for update;

* 意向锁：表级锁，申请数据库行锁时，需要申请相应的意向锁

  * 意向共享锁：表示申请数据库行的共享锁时，需申请意向共享锁
  * 意向排他锁：表示申请数据库行的排他锁时，需申请意向排他锁

  意向锁作用：如果没有意向锁，某行被加了排他锁，这时候申请该行所属表的排他锁，需要遍历所有行。如果引入意向锁，先申请到了意向排他锁，才能申请到行级排他锁，这时再请求表级别排他锁就会失败。

## 5.3 间隙锁和next key lock（RR级别生效）

InnoDB行锁有三种算法：

* record lock: 锁定索引记录本身
* gap lock: 在索引记录的间隙加锁，锁定一个范围，但是不包含记录本身
* next key lock: 锁定一个范围+记录本身(gap lock和record lock的结合)，可以防止幻读。

**举例**

|                                                   |
| ------------------------------------------------- |
| 事务1：select * from tbl where id > 16 for update |
| 事务2：insert into tbl (name) values ('test')     |

* 如果id有``10, 12, 29``，则RR级别下，next key lock的锁区间是：(负无穷,10],(11, 12],(12,29],(29,正无穷)，事务1会锁住(12,29],(29,正无穷)区间，事务2的insert会被block住，于是避免了幻读。
* 如果事务1为``select * from tbl where id=13 for update``，则[12,29)会被锁住。
## 5.4 乐观锁
乐观锁通过非阻塞的方式实现对共享资源的访问。
只支付一次：
``update order_tbl set pay_status=1 where id=1 and pay_status=0;``
可以用乐观锁实现对业务的加锁。
```
res = update lock_tbl set is_lock=true where is_lock=false and lock_id=xxx;
if (res == 0)
  return
执行业务 
update lock_tbl set is_lock=false where is_lock=true and lock_id=xxx;
```

错误使用：只插入一次：
``insert into xxx_tbl  ... where not exist ... limit 1``
注意这个limit 1是必须要带的，否则会重复插入。 FIX :如果真的并发插入了，也没法保证不重复。

## 5.5 Mysql死锁和解决方案

InnoDB可以自行解决死锁

## 5.5 MVCC

MVCC即多版本并发控制，每行数据保留多份。不需要锁就可以实现高级别的事务隔离。通过多版本解决**幻读**。

每个事务在执行第一条select时生成一个**read view**。read view包含：

* 未提交事务ID集合
* 最小未提交事务ID
* 最大事务ID

根据read review维护了如下事务水位，每个事务根据水位确定从undo log里读什么数据：

|**已提交事务**(如t1, t2, t3)|<u>未提交事务(t4,t7)+已提交事务(t5,t6)</u>|*未开始事务*(t8)|

1. 每次更新数据，undo log（**版本链**）里新增一条记录，记录当前事务ID，值。

2. 每次删除数据，undo log里复制一条最新记录，记录当前事务ID，标记为删除。

3. 查找数据时，从最新的记录开始找：

   * 如果事务的ID小于最小未提交的事务ID，表示该事务已经提交，返回这个数据。

   * 如果事务ID大于最大的事务ID，表示事务未开始，不可见。
   * 如果在最小未提交事务ID和最大事务ID之间(均包含)，如果这个事务ID在未提交事务ID里，则不可见，如果不在，则返回这个数据。



# 6. 分布式事务

分布式事务的常用方法：

* 消息队列：业务可以异步重试完成，https://www.cnblogs.com/rjzheng/p/10115798.html
* TCC事务模型：业务同步完成，https://www.cnblogs.com/rjzheng/p/10164667.html


# 7. Other

* ORM框架，数据库连接池：https://my.oschina.net/mengyuankan/blog/2664784
* JDBC事务：https://www.cnblogs.com/softidea/p/5877546.html ，https://www.marcobehler.com/guides/spring-transaction-management-transactional-in-depth
