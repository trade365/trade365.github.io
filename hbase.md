# Hbase

## 架构
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/0e6f3503f3ab911b937d8d5b4eaea942.webp?x-oss-process=image/format,png#pic_center)


HReginServer负责数据存储

一张表的逻辑视图

| rowkey | timestamp | base:name | base:age | contact:phone |
| ------ | --------- | --------- | -------- | ------------- |
| 1      | 1         | zhangsan  | 12       | 123           |
| 1      | 2         | zhangsan1 | 13       | 123           |
| 2      | 1         | lisi      | 10       | 345           |

冒号前是Column Family，冒号后是Qualifier，每个行都有相同的CF，但是每个CF的qualifier是多样的。rowkey是多版本的(timestamp标识，默认为写入Region Server的时间戳)，过期的版本会在compact时删除。

每个CF单独存储在StoreFile上，rowkey按照顺序分布在Region Server上，建议将rowkey打散，避免访问集中到固定几台Region Server，导致服务不可用。一种常用的方式是给rowkey增加md5前缀。

Hbase无法像redis一样提供单机高QPS的访问能力，如果有热key，建议使用本地缓存(如LRU)或者redis存放这些热key以缓解Hbase压力。

Hbase最小的存储单元为Block，默认为64K，如果一个rowkey的数据超过了64K可能会引起访问超时，即使不访问大的rowkey，也可能发生超时(访问相邻的rowkey？)

