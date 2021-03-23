Redis是目前最为主流的缓存技术之一，Redis基于内存操作从而拥有强大的性能，可以达到每秒10万次的请求，可以说是一款非常强大的缓存技术了。

本文分为三部分：

- 基础知识介绍
- 常用技术讲解与缓存机制
- 使用场景、缓存问题

# 基础知识介绍

**NoSQL概述**

什么是NoSQL？

NoSQL = Not Only SQL （不仅仅是SQL）

关系型数据库：表格 ，行 ，列

非关系型数据库：没有固定的查询语言，键值对存储，列存储，文档存储

随着web2.0互联网的诞生！传统的关系型数据库很难对付web2.0时代！尤其是超大规模的高并发的社区。



NoSQL 特点

1、方便扩展

2、大数据量高性能（Redis 一秒写8万次，读取11万）

3、数据类型是多样性的



NoSQL四大分类

KV键值对：如Redis主要是用于内容缓存，主要是为了处理大量数据高访问负载

文档型数据库：如MongoDBMongoDB 是一个基于分布式文件存储的数据库

列存储数据库：如HBase分布式文件系统，以列簇式存储，将同一列数据存储在一起

图关系数据库：如Neo4j他不是存图形，放的是关系，比如：朋友圈社交网络，广告推荐！



**Redis简介**

Redis 是什么？

Redis（Remote Dictionary Server )，即远程字典服务，它是一个开源的由ANSI C语言编写，性能优秀、支持网络、可持久化的Key-Value内存的NoSQL数据库!



Redis 能干嘛？

1、内存存储、持久化。

2、效率高，可以用于高速缓存

3、发布订阅系统

4、计时器、浏览量！

 5、........



**Redis好处**

主要从“高性能”和“高并发”这两点来介绍。

![Redis缓存机制与应用](https://p6-tt.byteimg.com/origin/pgc-image/7ede911b397f470f93ec818318fce4f6?from=pc)

把数据库数据存入缓存，请求直接从内存中读取不用经过数据库，减轻数据库压力并且提升性能。

# 常用技术讲解与缓存机制

Redis主要有5种数据类型，包括String，List，Set，Zset，Hash，满足大部分的使用要求

**String**

- String：session、对象、小文件（存文件流字节数组，比磁盘IO快）？
- int：秒杀、限流、计数
- bitmap：

场景1.setbit和bitcount结合可以统计一年365天哪天有用户操作过，getbit可以获取某一天是否用户操作过

场景2.权限控制，比如每个权限对应一个bit,哪个用户有该权限，该位为1，没权限为0

**list**

替换java jvm中的集合，可以作为数据共享，java的话多进程间不能共享或不好共享

**hash**

可以使redis key变少，类似对象。

场景1.商品详情页、商品对应的收藏数、库存啊，放在redis中因为是原子性的，多地方访问都是实时性的

场景2.聚合场景：一个对象在数据库中可能各个属性在不同表，可以聚合到redis同个对象中

**set**

set性能慢，可以单独redis实例

场景1.SRANDMEMBER或者spop命令可以用来抽奖

场景2.随机事件

场景3.共同好友（交集）

场景4.推荐好友（差集）

**sorted_set**

有序集合，数量少时底层是zipList压缩表，数据多了变skiplist

场景1.排行榜

场景2.有序事件

场景3.评论分页

![Redis缓存机制与应用](https://p3-tt.byteimg.com/origin/pgc-image/6ddde37802934a2b8e4a1717f732387e?from=pc)

Redis 中除开最常用的 5 种数据类型之外，还有 3 种特殊的数据类型

![Redis缓存机制与应用](https://p6-tt.byteimg.com/origin/pgc-image/b8e2cd5aec574ebab34e50602b9869a4?from=pc)

可以通过help命令查询相关类型命令说明，比如：

```
help @string help @list1
```

 

**事务**

Redis 事务本质：一组命令的集合！ 一个事务中的所有命令都会被序列化，在事务执行过程的中，会按照顺序执行！

Redis单条命令式保存原子性的，但是事务不保证原子性！

```
# 开启事务 
multi  

#命令入队 
set k1 v1 
set k2 v2   
get k2    


# 执行事务
exec   1
```

![Redis缓存机制与应用](https://p6-tt.byteimg.com/origin/pgc-image/44064f827927434aa3f4d7789a9249a6?from=pc)



**Redis持久化**

持久化就是把内存的数据写到磁盘中去，防止服务宕机了内存数据丢失。

redis提供两种持久化机制 RDB（默认） 和 AOF 机制。

1、RDB

RDB是Redis DataBase缩写快照 ，默认的持久化方式。按照一定的时间将内存的数据以快照的形式保存到硬盘中，对应产生的数据文件为dump.rdb

![Redis缓存机制与应用](https://p1-tt.byteimg.com/origin/pgc-image/deb121beff13408ab6532b014b07a7af?from=pc)

触发机制

（1）save的规则满足的情况下

（2）执行 flushall 命令

（3）退出redis，也会产生 rdb 文件



2、AOF：

持久化，AOF持久化(即Append Only File持久化)，则是将Redis执行的每次写命令记录到单独的日志文件中，当重启Redis会重新将持久化的日志中文件恢复数据。

![Redis缓存机制与应用](https://p3-tt.byteimg.com/origin/pgc-image/cb60ccda14e644a2872d5d25fcadd34b?from=pc)

AOF的三种策略（1）always （2）everysec(默认值) （3）no always



![Redis缓存机制与应用](https://p1-tt.byteimg.com/origin/pgc-image/6767598bca44402b89881933b24b16ff?from=pc)

在应用时，要根据自己的实际需求，选择RDB或者AOF，其实，如果想要数据足够安全，可以两种方式都开启，但两种持久化方式同时进行IO操作，会严重影响服务器性能，因此有时候不得不做出选择。



**redis主从复制**

概念主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(master/leader)，后者称为从节点(slave/follower)；数据的复制是单向的，只能由主节点到从节点。

优点：（1）读写分离 （2）备份

缺点：主服务器宕机，需要人工启动

![Redis缓存机制与应用](https://p1-tt.byteimg.com/origin/pgc-image/9297dd62499a485fb25f1c602f18f956?from=pc)


**哨兵模式**

哨兵模式是一种特殊的模式，首先Redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例。

![Redis缓存机制与应用](https://p3-tt.byteimg.com/origin/pgc-image/7b7765c3fc0c4523ae2193355945a56d?from=pc)



# 使用场景、缓存问题

1、热点数据的缓存

公司项目用户量达到一定数量的时候，这时合理的利用缓存不仅能够提升项目访问速度，还能大大降低数据库的压力。

2、业务上的统计，排行榜

为了保证数据实时效，比如项目的访问量，每次浏览都得给+1，并发量高时如果每次都请求数据库操作无疑是种挑战和压力

3、限时业务的运用

每日签到、限制登录功能等业务场景

4、消息队列

提供基本的发布订阅功能，但不像消息队列那种专业级别



**缓存雪崩**

原因：大量redis key在同一时间失效，导致大量请求访问数据库，数据库服务器宕机，线上服务大面积报错。

解决办法：

（1）redis高可用

（2）加锁排队，限流降级

（3）缓存失效时间均匀分布

![Redis缓存机制与应用](https://p1-tt.byteimg.com/origin/pgc-image/e387882067f24a498f52a740534e50e4?from=pc)



**缓存穿透**

原因：指缓存和数据库中都没有的数据，导致所有的请求都落到数据库上，造成数据库短时间内承受大量请求而崩掉。

解决办法： （1）接口层增加校验 （2）采用布隆过滤器

![Redis缓存机制与应用](https://p6-tt.byteimg.com/origin/pgc-image/9dce9ecc50fd4eb992babd6076787155?from=pc)

**缓存击穿**

原因：指缓存中没有但数据库中有的数据（一般是缓存时间到期），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力。比如微博热搜。

解决办法：

（1）设置热点数据缓存没有过期时间

（2）加互斥锁

![Redis缓存机制与应用](https://p6-tt.byteimg.com/origin/pgc-image/a5c4853f4b6249649895a970bb433d62?from=pc)



好了就介绍到这了，可以自己动手尝试吧。