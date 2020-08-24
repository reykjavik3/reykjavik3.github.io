---
layout: post
title: InnoDB中的锁机制解析2
categories: MySQL
description: InnerDB的锁
keywords: InnerDB,Lock
---

上次写了InnoDB解决幻读问题引入的间隙锁，也挖了binlog的坑，先补充几个小细节

# 小细节
1 索引上的等值查询，再给聚集索引加锁的时候，Next-key Lock退化为Record Lock
有的博客写的给唯一索引加锁Next-key Lock就会退化为Record Lock，我开了两个session尝试了，这个锁必须加在聚集索引上临键锁才会退化为行锁，看这个case
还是上篇那个表和间隙
(1,5],(5,10],(10,15],(15,20],(20,25];

|sessionA|sessionB|
|---|:---|
|begin;||
|select * from lk_t where item_id = 5 for update; ||
||insert into lk_t values(3,3,3,3) （block）|
这个例子中item_id是唯一索引，加上锁之后还是拿到了Next-key Lock 区间是 (1,5],(5,10);

如果锁加在聚集索引上就不一样了

|sessionA|sessionB|
|---|:---|
|begin;||
|select * from lk_t where id = 5 for update; ||
||insert into lk_t values(3,3,3,3) （Affected rows: 1）|

经过以上对比可以看出等值查询的退化条件是加在聚集索引上而不是唯一索引；

2 读锁中lock in share mode 和for update的区别
- lock in share mode加一个共享读锁， for update加一个排他读锁

在RR隔离级别下， 二者在读的时候都是区别于普通读的快照读，生成一个新的视图的当前读；

除此之外还有个细节
lock ini share mode 在索引覆盖的情况下，可以不锁主键，而for update 会认为你接下来要执行更新，所以会把主键也锁住

看下边的case

|sessionA|sessionB|
|---|:---|
|begin;||
|select id from lk_t where item_id = 5 lock in share mode;||
||begin;|
||select * from lk_t where id= 5 for update (返回结果)|

而如果sessionA改为for update ，那么sessionB会被阻塞

|sessionA|sessionB|
|---|:---|
|begin;||
|select id from lk_t where item_id = 5 for update;||
||begin;|
||select * from lk_t where id= 5 for update (block)|

3 第三个 在说一下为什么隔离级别的问题，有些公司采用的隔离级别是读提交RC，那么binlog就需要配置为row
这里把上次挖的坑填了
先说binlog的三种格式
- statement 记录原始的sql信息
- row 记录的是操作的行为
- mixed statement 和row的混合

在允许幻读的RC隔离级别下
看下边的case

|sessionA|sessionB|
|---|:---|
|begin;||
|select * from lk_t where gunan=5 for update;||
||insert into lk_t (6,6,6,5);|
|update lk_t set inv = 100 where gunan=5;||
|Time1||
||Time2|
|Time3||
sessionA可以在Time1提交，也可以在Time3提交，如果在time1提交 那么inv=100就只有id=5
如果在time3提交，那么SessionA的更新会印象sessionB插入的id =6的数据
正是这种不确定性，让statement格式的binlog在备库重放的时候带了不确定性而导致数据不一致
所以RC下要把binlog设置为row

row这么好，为什么要有第三种呢？第三种mixed是前两种混合，有时statement一个sql就能搞定，row要几千行
比如 update user set name = null where id<1000;
statement就一个sql够了，而row要记录影响的这999行数据，所以如果设置为mixid，InnoDB会自己判断是否会产生不一致，自己决定用哪种格式的binlog;

以上；

我们继续昨天的讲；

昨天写了InnoDB的Next_key Lock，和它的退化，今天说说表锁；



# 表锁
表锁有自增锁和意向锁

分开说

## 自增锁
``` sql
show create table lk_t
```
``` sql
CREATE TABLE `lk_t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `inv` int(11) DEFAULT NULL,
  `item_id` varchar(12) DEFAULT '0.00',
  `gunan` int(10) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `item` (`item_id`),
  KEY `gunan` (`gunan`)
) ENGINE=InnoDB AUTO_INCREMENT=154 DEFAULT CHARSET=utf8
```
可以看到最后一行
AUTO_INCREMENT=154
我的表里边数据并不连续 之前插入了一条id=153的数据又删了，我们做以下测试
``` sql
insert into lk_t values (200,200,200,200);
```
在show create table lk_t发现 AUTO_INCREMENT=201
那说明自增值不连续，且等于表中插入过的最大值+1，即使这条记录被删了，这个值也不会变；

> ps 这里自增值显示在建表语句中，但其实这个自增值并不是维护在表结构中，早起维护在内存，在MySQL8.0后才可以持久化，记录在redo log中，重启后靠redo log恢复重启前的值，这里再挖个坑把，后边聊重做日志的事；

看下边的case
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1aed9038cc14438ac5e42cb14949fcf~tplv-k3u1fbpfcp-zoom-1.image)
``` sql
insert into lk_t (inv,item_id,gunan)values(201,25,201);
```
由于iten_id是唯一索引，且=25的值已经存在，故本条sql冲突无法插入；
在执行show create table lk_t;
发现
AUTO_INCREMENT=202

还有一种就是事务回滚时，已经申请的主键值也不会释放，这里我就有个疑问，MySQL为什么没有把插入失败的自增值改回去呢？
其实是为了性能考虑
考虑这个case 
1 sessionA 申请id=201,sessionB申请id=202
那么AUTO_INCREMENT=203
2 sessionB提交，sessionA回滚
如果把sessionA申请的id=201退回去，接下来其他事务申请到id=2，在下次id=3，发现id=3已经存在

为了解决这个冲突，就需要每次申请时查询这个申请到的值是否存在，这样成本很高
另一种方法是把自增锁范围扩大，等事务提交在释放，那势必会影响并发性；

所以InnoDB设计为，语句执行失败也不回退id;

### 自增锁优化
MySQl5.1以前，自增锁会在执行后释放，5.1.22引入新策略
下面看自增锁优化说明
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd27af95fc5d44278a5227108859096c~tplv-k3u1fbpfcp-zoom-1.image)

发现说的并不是人话，我们翻译以下

MySQL关于自增锁的参数innodb_autoinc_lock_mode有三个值

1 这个值是0时，和老版本一样
2 这个值是1时，分两种情况
 - simple insert 会在自增锁申请之后马上释放
 - 而 bulk insert (比如insert...select) 执行前无法知道要申请多少个id,会在执行后释放
3 这个值是2时
 所有申请自增之间都在申请后释放
 为什么？？？
 insert...select有什么特别
 
 特别就特别在，执行前无法知道要申请多少个id
 
 看这个case
 
 |sessionA|sessionB|
 |---|:---|
 |insert into lk_t values(null,2,2,2);||
 |insert into lk_t values(null,3,3,3);||
  |insert into lk_t values(null,4,4,4);||
 ||create table lk_t2 like lk_t|
 |insert into lk_t values(null,6,6,6)|insert into lk_t2(inv,item_id,gunan) select inv,item_id,gunan from lk_t|
 
 如果sessionB申请后就释放可能是这样的
 
 sessionB插入两条记录
 （2，2，2，2），（3，3，3，3）
 然后sessionA申请后id=4
 之后id继续插入 （5，4，4，4）
 这时，如果binlog的格式是statement，不论先记录sessionA，还是sessionB，这个语句都会导致不一致的现象发生
 所以建议
 
 > innodb_autoinc_lock_mode=2 ，并且 binlog_format=row,既能提升并发性，又不会出现数据一致性问题
 
 另外，如果select...insert要插入一万行，那要申请一万次自增锁吗，答案是不用
 
 第一次会申请一个，第二次2个，第三次4个；
 
 ## 意向锁
 
 意向锁是什么，为什么要有意向锁？
 还是为了提高性能；
 
 为了支持不同粒度上进行的锁操作，InnoDB支持一种额外的锁方式，称之为意向锁，意向锁是将锁定的对象分为多个层次，意味着事务希望在更细粒度上加锁；
 如下图
 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/458f09ab65db4fdbb8e882392fd521a5~tplv-k3u1fbpfcp-zoom-1.image)
 若将上锁对象看成一棵树，那么对最下层的对象上锁，要对上层的粗粒度对象上锁，比如要在行上加X锁，那么需要分别对记录所在DB，table,页分别上意向IX锁，最后对记录上X锁
 比如，在对记录r加X锁之前，已经有事物对表1进行了S表锁，那么表1上已经存在s锁，之后事物需要对记录r在表1上加IX，由于不兼容，所以钙食物需要等待表锁操作的完成
 - 意向共享锁 ，事务想要获得一张表中的某几行共享锁
 - 意向排他锁，事务想要获得一张表中的某几行排他锁
 
 ||IS|IX|S|X|
 |---|:---|:---|:---|:---|
 |IS|兼容| 兼容|兼容|不兼容|
 |IX|兼容|兼容|不兼容|不兼容|
 |S|兼容|不兼容|兼容|不兼容|
 |X|不兼容|不兼容|不兼容|不兼容|
 
 这时候 我又有个疑问了
 
IX 与 X冲突，那岂不是任意两个写操作，即使写不同行也会造成死锁
比如：
Session A 请求 IX--成功；
Session B请求 IX--成功
Session A 请求 X，发现已经有其他session有IX，因此冲突同理SessionB请求X也会是这种情况。那这row lock还有什么用？

可是实际情况不是这样，SessionA和SessionB写不同行是可以成功的并不会死锁；

原来
- IX，IS是表级锁，不会和行级的X，S锁发生冲突。只会和表级的X，S发生冲突
- 行级别的X和S按照普通的共享、排他规则即可。所以之前的示例中第2步不会冲突，只要写操作不是同一行，就不会发生冲突。

那解答了这个疑问 我们在向意向锁是干啥的；

当再向一个表添加表级X锁的时候如果没有意向锁的话，则需要遍历所有整个表判断是否有行锁的存在，以免发生冲突如果有了意向锁，只需要判断该意向锁与即将添加的表级锁是否兼容即可。因为意向锁的存在代表了，有行级锁的存在或者即将有行级锁的存在。因而无需遍历整个表，即可获取结果

总结下来就是避面过多判断，增加效率用的；


# InnoDB的锁结构是怎样的，怎么加的

InnoDB的锁粒度最小在行锁，如果对每一行维护一个锁对象，那么效率不高，实际上，InnoDB支持行锁，同时锁的开销很小，锁对象的管理通过位图的方式实现，先看下行锁在InnoDB中的定义
``` c
struct lock_rec_struct{
 ulint space; /*表空间编号*/
 ulint page_no;/*数据页编号 */
 ulint n_bits;/*数据页大小*/
 byte   bitmap[1+n_bits/8];
}
```
在看表锁的结构体
``` c
struct lock_table_stuct{
	dict_table_t* table;
    UT_LIST_NODE_T(lock_t) locks;
}
```

在这个锁结构中我们无法知道某个锁是共享的还是排他的，因为共享还是排他是定义在事务中 ；

每个事务会有一个对应的锁结构
看下边这个和事务有关的结构体

``` c
struct lock_struct{
	trx_t* trx; /* transaction owning the lock */
    UT_LIST_NODE_T(lock_t) trx_locks; /* list of the lock the transaction*/
    ulint type_mode /* lock type,mode,gap flag,andwait*/
    hash_node_t hash 
    dict_index_t* index /* index for a record lock*/
    union{
    lock_table_t tab_lock;
    lock_rec_t rec_lock;
    } un_member; /*共用体*/
};
```
想快速知道某个记录是否有锁，InnoDB通过这个结构体来达到目的
``` c
struct lock_sys_struct{
	hash_table_t* rec_hash; /* hash table of the recode locks*/
 }
```
 
需要先根据页space和page_no所处hash表中的键值，在根据记录所在页锁秒lock bitmap，才能知道是否有锁；这种方式使锁的开销会非常小，除了锁本身的开销之外，不需要而外的开销，如果根据一个页中每个记录进行锁管理，那么开销会非常巨大

# 死锁

## 两阶段协议
InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束才释放，这个就是两阶段协议。

## 定义和死锁检测
当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致 这几个线程都进入无限等待的状态，称为死锁。
看下边的case

|sessionA|sessionB|
|---|---|
|begin;||
|update lk_t set inv = inv+1 where id=1|begin;|
||update lk_t set inv = inv+1 where id=2; |
|update lk_t set inv = inv+1 where id=2;||
||update lk_t set inv = inv+1 where id=1;|

这时候sessionA在等session释放id=2的锁，而sessionB在等sessionA释放id=1的锁；两个session互相等待，就进入了死锁状态

两种策略解决死锁问题
- 直接进入等待，直到超时，这个超时时间可以通过innodb_lock_wait_timeout来配置
- 发起死锁检测，发现死锁后，主动回滚死锁链中的莫哥事务，参数innodb_deadlock_detect设置为on，表示开启这个逻辑；

正常情况下 我们采用第二种策略，InnoDB采用DFS的遍历方式，检测事务链表是否有环；


## 如何避免死锁
1 加锁的先后顺序尽量保证一致
2 一次性申请足够的锁

对于2 ，考虑这个case

|sessionA|sessionB|
|---|---|
|begin;|begin;|
|select * from lk_t where id=5 lock in share mode;||
||select * from lk_t where id=5 lock in share mode;|
|update lk_t set inv =6 where id =5;||
||update lk_t set inv =6 where id =5;(1213 - Deadlock found when trying to get lock; try restarting transaction, Time: 0.011000s)|
sessionB检测到死锁回滚，sessionA拿到id=5的写锁，执行成功
这个例子中两个session都是先加了共享锁，再加排他锁，正确的做法是查询时的lock in share mode换为for update,而且for update的语意就是将要执行更行；

3 大事务提高了死锁的纪律，尽量用小事务

## 如何减小行锁的影响
1 和上一篇说的一样，把一行数据拆分为多行，比如同商品的库存，原来商品id为1的库存为100，我现在让商品id可以不唯一，拆分为5行，每行库存为20，扣减库存时随机拿出一行数据；

2 如果一个事务中有插入，有更新，更新会拿到读锁，由于两阶段提交协议，锁是在事务提交后才释放的，所以尽量把update才做放到事务的后边，减小锁持有的时间

# 小结
今天承接上一篇，和大家聊了InnoDB加个行锁的小细节，聊了表锁中的自增锁和意向锁，之后我们学习了InnoDB锁的具体表现形式；
最后我们学习了死锁的定义和如何避免死锁以及如何减小行锁的影响

我们下期见！



