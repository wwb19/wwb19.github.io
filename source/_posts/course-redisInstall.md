---
title:  【教程】Redis 3.2 详细安装步骤说明
date: 2018-06-18 18:41:03
tags: 
   - 技术 
   - redis 
   - 教程
   - 生产
categories:
    - 技术
    - 教程
copyright: true
---


# Note
>本文用于Redis的部署安装
>配置文件内容需根据实际情况进行设置
>本文为Sentinel哨兵模式的集群搭建，若需Cluster集群模式，可查看Redis安装部署内容，集群安装不适合。
<!-- more --> 
# 一、部署说明

## 集群信息

* 集群：1主2从

* master：主服务器

* slave：从服务器

同步时间间隔：1s。每秒中，slave从master进行数据同步操作。

## 硬件配置要求

|配置项|内容|
| --- | --- | 
| CPU | 4c\*2.4g | 
| 内存 | 32g |
| 磁盘 | 1000G |
| 系统版本 | Redhat 7.3 |
| Redis版本 | 3.2.11 |

* 哨兵集群：Redis-Sentiel

相互服务发现机制，通过选举确定master是否可用。

选举配置：2。意味着3台中只要有两台无法与master通讯时，进行切换master操作。

* 读写分离

Master作为写服务器，Salve作为读服务器。通过数据同步，保证主从数据一致。

# 二、安装部署

## 1、redis安装部署步骤

### Master节点安装

#### 1、安装系统依赖包
```
yum install --downloadonly --downloaddir=/tmp/download
```

```
yum -y install gcc automake autoconf libtool make telnet ruby-devel ruby-irb ruby-libs ruby-rdoc ruby rubygems-devel rubygems
```
#### 2、安装ruby包
```
yum -y install centos-release-scl-rh centos-release-scl

sed -i -e s/\]$/\]\npriority=10/g /etc/yum.repos.d/CentOS-SCLo-scl.repo

sed -i -e s/\]$/\]\npriority=10/g /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo

sed -i -e s/enabled=1/enabled=0/g /etc/yum.repos.d/CentOS-SCLo-scl.repo

sed -i -e s/enabled=1/enabled=0/g /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo

yum --enablerepo=centos-sclo-rh -y install rh-ruby22

scl enable rh-ruby22 bash

ruby –v
```

#### 3、安装reids ruby控制包

```
gem install redis
```

#### 4、安装redis
```
cd  /tmp

yum install -y wget

wget http://download.redis.io/releases/redis-3.2.11.tar.gz

tar -zxf redis-3.2.11.tar.gz

cd redis-3.2.11

make

cd src &amp;&amp; make install PREFIX=/usr/local/redis

cd ..

mkdir /usr/local/redis/etc

mkdir /usr/local/redis/db

cp redis.conf /usr/local/redis/etc/

cp ./src/redis-trib.rb /usr/local/redis/bin/
```
#### 5、修改配置文件
```
vi /usr/local/redis/etc/redis.conf

#bind 127.0.0.1 #这里注释掉，否则会导致其它节点无法与本节点通信

protected-mode yes

daemonize yes #用守护模式启动

pidfile /var/run/redis\_6379.pid #不同机器PID文件不一样，我们用7001-7006 #可以用同一个，但是为了统一还是建议改成不同的

logfile /var/log/redis.log

dir /usr/local/redis/db/

masterauth password

requirepass password

appendonly yes #开启AOF模式
```

### Slave节点安装

#### 同上1-4

#### 修改配置文件
```
vi /usr/local/redis/etc/redis.conf

#bind 127.0.0.1 #这里注释掉，否则会导致其它节点无法与本节点通信

protected-mode yes

daemonize yes #用守护模式启动

port 6379 #修改为port 6380

#增加 slaveof 192.168.81.129 6380

slaveof 172.20.10.12 6379

pidfile /var/run/redis\_6380.pid #不同机器PID文件不一样， #可以用同一个，但是为了统一还是建议改成不同的

logfile /var/log/redis.log

dir /usr/local/redis/db/

masterauth password

requirepass password

appendonly yes #开启AOF模式
```
增加requirepass password，这是设置redis客户端或者远程机器连接redis服务器需要的密码，这里同样设为redis（6381的密码也设为redis）

### Sential哨兵安装

#### 安装哨兵

哨兵与redis在同一服务器下，无需安装，redis自带。

若与redis不在统一服务器下，可参考上redis安装1-4步骤。

#### 修改配置（每个哨兵配置，除监听端口外，其他保持一致）
```
cd /tmp/redis-3.2.11
```
复制配置到 etc目录

```
cp sentinel.conf /usr/local/redis/etc/sentinel.conf

vi /usr/local/redis/etc/sentinel.conf

增加

bind 0.0.0.0

端口是 26379

增加：

daemonize yes

logfile /var/log/sentinel.log

搜索 sentinel monitor  下面增加

sentinel monitor mymaster 172.20.10.12 6379 2

对sentinel monitor mymaster 192.168.81.129 6379 2
```

解释：
mymaster ：为主服务器起的名字；

192.168.81.129 6379：主服务器的ip和端口号

2：代表有2个哨兵认为主服务器主观下线时，则认为主服务器是客观下线了，可以执行主从切换，并进行故障转移操作

```
搜索 auth-pass  下面增加ps
sentinel auth-pass mymaster password
```

## 2、集群启动

### 启动redis集群：

#### 服务器A：

redis – 6379

哨兵-26379

/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf

#### 服务器B：

redis – 6380

哨兵-26380

/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf

#### 服务器C：

redis – 6381

哨兵-26381

/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf

### 启动哨兵集群：

#### 服务器A：

/usr/local/redis/bin/redis-sentinel /usr/local/redis/etc/sentinel.conf

#### 服务器B：

/usr/local/redis/bin/redis-sentinel /usr/local/redis/etc/sentinel.conf

#### 服务器C：

/usr/local/redis/bin/redis-sentinel /usr/local/redis/etc/sentinel.conf



## 3、安装验证

### 集群部署验证
```
/usr/local/redis/bin/redis-cli

  127.0.0.1:6379 auth password

  OK

  127.0.0.1:6379 info replication

# Replication

role:master

connected\_slaves:2

slave0:ip=172.17.0.1,port=6380,state=online,offset=407,lag=1

slave1:ip=172.17.0.1,port=6381,state=online,offset=407,lag=1

master\_repl\_offset:407

repl\_backlog\_active:1

repl\_backlog\_size:1048576

repl\_backlog\_first\_byte\_offset:2

repl\_backlog\_histlen:406
```

#### 查询验证

```
/usr/local/redis/bin/redis-cli -h 172.20.10.12 -p 6380

  127.0.0.1:6380 auth password

  OK

  127.0.0.1:6380 keys \*

  (empty list or set)
```
  
#### 新增验证

无

