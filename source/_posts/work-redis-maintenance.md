work-redis-maintenance

---
title:  【培训】Redis日常运维
date: 2018-09-06 20:24
tags: 
    - 培训 
    - redis 
    - 技术
    - 会议
    - 运维
categories:
    - 工作
    - 培训
copyright: true
---

# Note
>本文介绍Redis的日常运维
>本文为Sentinel哨兵模式，若需Cluster集群模式，有部分内容不适用

# 大纲
* [Redis键值操作](#command)
* [Redis监控](#monitor)
* [Redis哨兵重启](#restart)
* [Redis故障恢复](#failure)
* [总结](#summary)

<a id="command"></a>
# Redis键值操作
## Hash
* 获取key

```
hgetall key
```
***PS:大数据量的key，绝对不能在生产上执行***

* 获取key和域

```
hget key field
```

* 获取大小

```
hlen key
```

## Str
* 获取key（不能执行）

```
keys *
```
***绝对不能在生产上执行***

* 获取key对应的值

```
get key
```

* 获取整个Redis的key总数

```
dbsize
```

## Set

* 获取整set的数量

```
smembers key
```

## List

* 获取整list的数量

```
llen key
```

<a id="monitor"></a>
# Redis监控
## 整体信息

```
redis-cli -p [PORT] -h [IP] -a [PASSWORD] info all
```
1、主要看看到所有的Redis当前运行情况，当Redis发生问题时，需把整个信息保存下来。
2、此信息有需要的话，可以每隔一定时间间隔打印到一个日志文件中，以便于监控Redis的运行情况。
3、此命令与下面介绍的命令一样，只是all显示所有的信息。后续会针对于重点信息介绍。

## 集群信息

```
redis-cli -p [PORT] -h [IP] -a [PASSWORD] info server 
```
可以看到当前哨兵集群的运行情况。
1、可以知道当前服务器的角色，是slave还是master
2、获取集群中其他的服务器信息或主节点的地址和端口
3、本服务器的端口
4、redis的版本信息以及其他配置信息

## 内存使用情况
```
redis-cli -p [PORT] -h [IP] -a [PASSWORD] info Memory 
```
可以获取当前Redis内存使用情况
1、used_memory_human--可以知道当前内存使用情况。
2、maxmemory_human--配置的最大内存。（若涉及持久化的话，一般会配置为本机内存的60%，其他情况下根据实际情况配置）
3、mem_fragmentation_ratio -- 内存碎片率，重要关键信息，若小1，则必须新增服务器内存；若大于1.5，则需要重启Redis以释放内存。

## 命令执行情况
```
redis-cli -p [PORT] -h [IP] -a [PASSWORD] info Commandstats 
```
这个命令可以看出Redis有哪些命令执行过，执行的耗时多少，结合进行分析当前Redis运行情况。是否存在慢操作


## 主从同步
```
redis-cli -p [PORT] -h [IP] -a [PASSWORD] info Persistence 
```
这个可以看出主从同步的情况。
1、rdb_last_bgsave_status--可以看出当前bgsave是否正常，一般为ok。若存在异常情况，会导致Redis无法写入。
2、rdb_last_bgsave_time_sec--上一次bgsave执行耗时，可以通过这个命令知道Redis执行bgsave的耗时情况。

## TPS
```
redis-cli -p [PORT] -h [IP] -a [PASSWORD] --stat 
```
这个命令在压力测试时可以使用。用来观察当前Redis的TPS压力情况。一般Redis的TPS可以达到10W+，这个可以知道当前的Redis服务器是否已经处于比较高的TPS导致无法接收更多指令。（一般10W以内都可以接受）

## cpu
```
top
nmon
vmstat
```
直接通过常用top命令或者nmon观察cpu消耗。
由于Redis属于单线程操作内存数据，因此会发现在单个CPU上有可能存在100%的情况，这个也需要关注，若持续在一个单核上都是100%，说明存在Redis命令执行过长的情况。一般都会随机到不同的cpu执行。相对平均。

## 慢日志
```
redis-cli -p [PORT] -h [IP] -a [PASSWORD] 

>slowlog get
```
可以知道当前Redis最大的耗时操作有哪些。特别进行关注。
一般操作超过100微秒都可以重点关注，除了一些复杂的lua脚本之外，但是不建议在Redis内进行复杂的lua操作。



<a id="restart"></a>
# Redis哨兵重启
## 停止集群
### 1、持久化数据
Redis若没有配置持久化，重启后，内存中的数据会全部丢失。
因此根据需要，考虑是否在哨兵集群重启前进行bgsave操作，避免数据重新加载增加交易耗时。
使用bgsave后，在Redis的服务器上会生成对应的内存rdb快照文件，待Redis启动时会读取这个文件的内容，恢复至内存中。
**PS：避免存在脏读的情况，因此若要求每次都重新加载则不能执行bgsave操作**
```
redis-cli -p [PORT] -h [IP] -a [PASSWORD] bgsave
```

### 2、停止哨兵集群
由于哨兵集群用于监控Redis服务器的运行情况，重启前需先把哨兵停止，避免因此导致的主从切换
```
redis-cli -p [Sentinel PORT] -h [Sentinel IP] shutdown
```

### 3、停止Redis集群
由于涉及主从切换，当主服务先停止则从服务器无法获取主服务器信息，可能会设置自身状态为down，导致下次重启时触发全量同步，因此建议优先停止从服务器。
* （1）、停止slave从服务器

```
redis-cli -p [Slave PORT] -h [Slave IP] shutdown
```
* （2）、停止master主服务器

```
redis-cli -p [Master PORT] -h [Master IP] shutdown
```

## 启动集群
原则：先启动主服务器，再启动从服务器，当整个Redis集群，起码Master服务器状态为正常后，再启动哨兵服务器，对外提供服务。
### 1、启动主服务器
```
redis-server conf/redis.conf
```
### 2、启动从服务器
```
redis-server conf/redis.conf
```

### 3、启动哨兵服务
```
redis-sentinel conf/sentinel.conf
```

<a id="failure"></a>
# Redis故障恢复
## 主从切换
当主服务器无法正常对外提供服务后，哨兵模式下会自动触发主从切换，哨兵最迟会在30s发现主服务器无法访问，通过选举方式，设置其中一台从服务器为主服务器。达到主从自动切换的目的。

主从切换可能会引起一些情况：

* 主从切换，会触发主从同步，若全量同步会触发master进行bgsave操作，导致cpu和本地IO升高。此时还有大量的CPU消耗的话，会导致Redis无法获取CPU资源，导致Redis长时间无法使用，因此建议避免多个Redis实例在同一台机器上都作为master节点使用，也避免在服务器部署多个独立的Redis实例应用在不同的业务系统导致相互影响。
* 主从切换，会触发主从同步，若全量同步此时会有可能会导致网络带宽消耗比较大。

## 删除大key
由于Redis属于单线程操作，因此堵塞对于Redis来说是致命的。避免大key的操作很重要。
1、避免设置大key自动回收或者时限。
2、避免在线进行大key的直接delete，可使用scan方式，或者例如 set中pop方式进行清理。清理后，内存碎片率偏高，需定期重启Redis集群释放。
3、避免使用过大的key，进行通过不同的方式减少key的大小。

## 内存不足
随着业务的扩张，Redis中的数据若无法进行清理，则需扩展内存。
扩展内存无需重启Redis器，但是要重要的是，若Redis配置中已经设置了最大内存，则需重新设置，否则不生效。
当然也可以使用Redis自身的清理机制，但需结合业务考虑，另外Redis自身清理也可能引发堵塞，这个要重点关注。因此一般默认配置为Redis内存满了，无法提供服务。

## 重复主从同步
这个由于并发量太大，再加上同步缓存冲设置太小导致的。
会严重占据Master的单个cpu以服务器的带宽，若多台slave同时请求同步，会更严重。
因此需通过参数调整优化，根据业务量来评估设置缓存区多少，默认为1m，当前我们项目使用为256m。


<a id="summary"></a>
# 总结
目前Redis已经属于成熟的运行平台和行业方案，稳定性很有保障。不过对于Redis的用法还是需要多注意，特别几点 **阻塞命令操作、大key删除、monitor监控、多实例争夺、主从同步等**。
日常运维主要关于是**内存使用率、内存碎片、命令平均耗时、CPU耗时等**
若有说的不对的或者不完善的，可联系。相互学习。


