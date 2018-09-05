---
title:  【培训】公司Redis分享会
date: 2018-06-22 17:19:03
tags: 
    - 培训 
    - redis 
    - 技术
    - 会议
categories:
    - 工作
    - 培训
copyright: true
---

# 大纲
* [Redis介绍](#overview)
* [Redis使用](#useage)
* [Redis场景](#stage)
* [Redis实践](#practice)
<!-- more --> 
# 一、Redis介绍
<a id="overview"></a>
## 概述
Remote DIctionary Server(Redis) 是一个由Salvatore Sanfilippo写的key-value存储系统。

Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。

它通常被称为数据结构服务器，因为值（value）可以是 字符串(String), 哈希(Map), 列表(list), 集合(sets) 和 有序集合(sorted sets)等类型。
## 特性
* Redis支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。

* Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。

* Redis支持数据的备份，即master-slave模式的数据备份。

## 数据结构
### Redis支持五种数据类型：
```
//type的占5种类型：
/* Object types */
#define OBJ_STRING 0    //字符串对象
#define OBJ_LIST 1      //列表对象
#define OBJ_SET 2       //集合对象
#define OBJ_ZSET 3      //有序集合对象
#define OBJ_HASH 4      //哈希对象
```

### Redis底层的十种基础数据结构（3.2）
```
// encoding 的10种类型
#define OBJ_ENCODING_RAW 0          //原始表示方式，字符串对象是简单动态字符串
#define OBJ_ENCODING_INT 1          //long类型的整数
#define OBJ_ENCODING_HT 2           //字典
#define OBJ_ENCODING_ZIPMAP 3       //不在使用
#define OBJ_ENCODING_LINKEDLIST 4   //双端链表,不在使用
#define OBJ_ENCODING_ZIPLIST 5      //压缩列表
#define OBJ_ENCODING_INTSET 6       //整数集合
#define OBJ_ENCODING_SKIPLIST 7     //跳跃表和字典
#define OBJ_ENCODING_EMBSTR 8       //embstr编码的简单动态字符串
#define OBJ_ENCODING_QUICKLIST 9    //由压缩列表组成的双向列表-->快速列表
```

### 相互关系
|  数据类型  |             基础数据结构 |
|----------|-------------------------|
|String    |INT,EMBSTR,RAW           |
|LIST      |ZIPLIST,QUICKLIST        |
|HASH      |ZIPLIST,HT(HASHTABLE)    |
|SET       |INTSET,HT                |
|ZSET      |ZIPLIST,SKIPLIST         |

## 优势
* 性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s 。

* 丰富的数据类型 – Redis支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作。

* 原子 – Redis的所有操作都是原子性的，意思就是要么成功执行要么失败完全不执行。单个操作是原子性的。多个操作也支持事务，即原子性，通过MULTI和EXEC指令包起来。

* 丰富的特性 – Redis还支持 publish/subscribe, 通知, key 过期等等特性。

# 二、Redis使用
<a id="useage"></a>
## 安装方式
请参考另一篇文章{% post_link course-redisInstall [Redis 3.2 详细安装步骤说明]%}
## 常用命令
[常用命令介绍](https://www.cnblogs.com/kevinws/p/6281395.html)
## 持久化介绍
Redis 分别提供了 RDB 和 AOF 两种持久化机制： 

* RDB 
将数据库的快照（snapshot）以二进制的方式保存到磁盘中。
* AOF 
则以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到 AOF 文件，以此达到记录数据库状态的目的。

## 高可用
### 主从同步

### 哨兵模式
Sentinel（哨兵）是Redis 的高可用性解决方案：由一个或多个Sentinel 实例 组成的Sentinel 系统可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器。
参考部署架构图

* 正常状态下：（A-主，B、C为从）
![](http://oxigzlivh.bkt.clouddn.com/2018-06-23-15297206515518.jpg)

* Master掉线后：
![](http://oxigzlivh.bkt.clouddn.com/2018-06-23-15297206846504.jpg)

* 选举升级Slave为Master：
![](http://oxigzlivh.bkt.clouddn.com/2018-06-23-15297207006886.jpg)

* 恢复正常后：
![](http://oxigzlivh.bkt.clouddn.com/2018-06-23-15297207149684.jpg)


# 三、Redis场景
<a id="stage"></a>
## 表数据缓存
使用方式：直接把表数据加载至Redis中，数据类型为Hash，常用的数据库缓存的用法之一。

* 一般用法：

```
如：
 **表名：**TableName
 **字段：**A,B,C,D
 转换为Redis的存为：(结构为hash)
 **Key：**T_TableName_唯一索引
 **Field：**
 {a:val1,b:val2,c:val3,d:val4}
```
 
* 说明：
此类一般保持少量并且常用的字段加载到内存中进行缓存使用，因此需要保证Field的数据量条数不多并且val字段长度不大，尽量使用Ziplist以便节省内存空间。
## NoSQL数据库
使用方式：直接以Redis作为数据库存储，可以同表数据缓存方式进行存放，或重新定义所需的数据结构。

* 一般用法：

```
如：对象User

Key为：User:uid
Field：对应属性字段
或
Key为：User
Field:uid.属性字段
```

* 说明：
需要进行数据量的评估和考虑数据结构对于内存的使用。
因为在Redis中，若所有数据均以Key-Vale的方式存放，每个key除了自身数据外，还需保留RedisObeject本身的数据空间，因此占用较大，可考虑转为Hash-Field的方式进行数据存放。
但Hash使用两种数据结构，一个是ZipList一个是HashTable，若使用HashTable方式存放，则内存空间无法节省，因此需结合具体情况定义不同的数据结构使用方式。

## 消息队列（生产者消费模式）
Redis不仅可作为缓存服务器，还可用作消息队列。它的列表类型天生支持用作消息队列。
Redis提供了两种方式来作消息队列。 
一个是使用生产者消费模式，另一个就是发布订阅者模式。 
前者会让一个或者多个客户端监听消息队列，一旦消息到达，消费者马上消费，谁先抢到算谁的，如果队列里没有消息，则消费者继续监听。 
后者也是一个或多个客户端订阅消息频道，只要发布者发布消息，所有订阅者都能收到消息，订阅者都是平等的。
如下图所示：

![](http://oxigzlivh.bkt.clouddn.com/2018-06-23-15297224494777.jpg)

由于Redis的列表是使用双向链表实现的，保存了头尾节点，所以在列表头尾两边插取元素都是非常快的。所以可以直接使用Redis的List实现消息队列，只需简单的两个指令lpush和rpop或者rpush和lpop。

## 流水号判重
Redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。
Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。
集合中最大的成员数为 232 - 1 (4294967295, 每个集合可存储40多亿个成员)。
我们可以使用Set进行**流水号判重操作**

```
如：
Key:serno_YYYYMMDD
Set:[serno1,serno2 ... sernoN]

Command：
SADD serno_YYYYMMDD sernoX 
加入Set中，若已存在则返回0，若加入成功则返回1
```

# 四、Redis实践
<a id="practice"></a>
## 支付系统-额度服务
请参考另一篇文章{% post_link practice-redis-s00 [【实践】记Redis使用实践 ]%}
## Redis数据迁移
方案进行中，后续分享。

 

