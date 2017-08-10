---
title: Redis集群安装以及基本使用
tags: redis
top: 0
---

---
Redis作为缓存系统来说还是很有价值的，在大数据方向里，也是需要有缓存系统的。一般可以考虑tachyon或者redis，由于redis安装以及使用更简单，所以还是优先考虑了它。那么在一些场景下为了保证数据的可靠性，就需要采用集群的模式部署，因此本篇文章就基于Redis Cluster的背景讲解下部署以及后期的使用。
<!-- more -->

大致会包括下面的内容：

- Redis单机版的安装以及验证
- Redis集群版的安装以及验证
- 使用图形化工具访问Redis
- 使用Jedis访问Redis
- 使用JedisCluster访问Redis Cluster

之后会介绍一下，如何在Spark中使用redis，敬请期待。

## Redis单机版安装

#### 1.1 下载
首先去官网下载想要的版本：
https://redis.io/download

我这里选了一个版本没那么高的，省的变化太大，各种软件兼容不了。于是挑个之前的稳定版本下载，下载`redis-3.2.10.tar.gz`拷贝到目标机器。

#### 1.2 安装

把压缩文件拷贝到指定的服务器上，执行解压命令：
```
tar -zxvf redis-3.2.10.tar.gz
```
安装前需要先安装必要的包，`yum -y install gcc gcc-c++ tcl`，这里安装的都是一般遇到的错误所需要安装的库。

安装完后，进入`redis-3.2.10`目录，执行编译命令：`make MALLOC=libc`，这样就安装完了

#### 1.3 测试

安装完可以直接启动测试一下：

- 启动服务器，执行下面的命令：`./src/redis-server redis.conf`
- 启动客户端，执行下面的命令：`./src/redis-cli`

在控制台中，输入下面命令:
```
set foo bar
get foo
```
看看是否正常

![](http://images2017.cnblogs.com/blog/449064/201708/449064-20170809194425824-343033669.png)


## redis集群安装

按照上面的方式，在每台机器上面都装一下redis。由于redis cluster集群必须要三个master，如果做一个备份的话，就需要6个节点。所以我这里在三台机器上，安装redis，然后每台机器上启动两个redis即可。一会会说一下，如何在一台机器上，启动两个redis。

#### 2.1 安装

首先安装一下ruby，因为集群的脚本是ruby版的
```
yum -y install ruby ruby-devel rubygems rpm-build
```
然后执行：
```
gem install redis
```
然后把`redis-trip.rb`拷贝到`/usr/local/bin`下，这样就可以在任何目录执行`redis-trip`命令了。
```
cp src/redis-trip.rb /usr/local/bin
```

#### 2.2 修改配置

接下来需要根据集群的情况，创建server启动的配置了，先创建`redis_cluster`目录，拷贝`redis.conf`：
```
[root@localnode7 redis-3.2.10]# mkdir redis_cluster
[root@localnode7 redis-3.2.10]# cp redis.conf ./redis_cluster/redis.conf
```
然后修改redis.conf
```
[root@localnode7 redis-3.2.10]# vi ./redis_cluster/redis.conf
```
修改的内容有：
```
bind 0.0.0.0 //为了别人能访问，这里暴力的监听了所有的地址
port 6379
daemonize yes //redis后台运行
cluster-enabled yes //开启集群  把注释#去掉
cluster-config-file nodes_6379.conf //集群的配置  配置文件首次启动自动生成
cluster-node-timeout 15000 //请求超时  默认15秒，可自行设置
appendonly yes //这里跟redis持久化的机制有关系，有兴趣可以多关注一下redis的日志记录机制 
```
PS：因为`redis.conf`配置文件比较大，内容很多。因此大家在修改配置的时候可以使用vi的快捷键`/想要搜索的内容`，然后按`enter`,可以快速定位到想要修改的配置上，如果有多个，可以再用`n`键跳转到下一个匹配结果。

#### 2.3 启动服务器

接下来就可以启动redis服务器
```
[root@localnode7 redis-3.2.10]# ./src/redis-server redis_cluster/redis.conf
[root@localnode7 redis-3.2.10]# ps -aux | grep redis
Warning: bad syntax, perhaps a bogus '-'? See /usr/share/doc/procps-3.2.8/FAQ
root     29361  0.0  0.0 130436  2452 ?        Ssl  13:21   0:00 ./src/redis-server 127.0.0.1:6379 [cluster]
root     29381  0.0  0.0 103464  1080 pts/0    R+   13:21   0:00 grep redis
[root@localnode7 redis-3.2.10]#
```
#### 2.4 单个机器启动多个server

如果想要在一台机器上同时启动多个redis，可以直接再拷贝一份redis.conf，命名成redis2.conf，修改里面的`port`参数和`cluster-config-file`参数即可。

比如上面的配置修改成:
```
bind 0.0.0.0
port 6380  // <----只有这里发生变化
daemonize yes
cluster-enabled yes
cluster-config-file nodes_6380.conf // <--只有这里发生变化
cluster-node-timeout 15000
appendonly yes
```
然后启动的时候，指定为这个conf即可:
```
./src/redis-server redis_cluster/redis2.conf
```

再次查询启动进程：
```
[root@localnode4 redis-3.2.10]# ps -aux | grep redis
Warning: bad syntax, perhaps a bogus '-'? See /usr/share/doc/procps-3.2.8/FAQ
root      7452  0.0  0.0 131892  2984 ?        Ssl  15:59   0:08 ./src/redis-server 0.0.0.0:6379 [cluster]
root      9337  0.0  0.0 130564  2744 ?        Ssl  16:00   0:08 ./src/redis-server 0.0.0.0:6380 [cluster]
root     32298  0.0  0.0 103464  1084 pts/0    S+   19:27   0:00 grep redis
[root@localnode4 redis-3.2.10]#
```

#### 2.5 创建集群

注意：

- 想要搭建集群，至少6个节点，不然执行下面的命令会报错的
- 在创建集群时，只能用ip地址，不能用主机名，不然会连接不上...不知道为啥...

执行下面的命令启动集群创建：
```
[root@localnode6 redis-3.2.10]# redis-trib.rb  create  --replicas 1 10.10.10.104:6379 10.10.10.106:6379 10.10.10.107:6379 10.10.10.104:6380 10.
10.10.106:6380 10.10.10.107:6380
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
10.10.10.107:6379
10.10.10.106:6379
10.10.10.104:6379
Adding replica 10.10.10.106:6380 to 10.10.10.107:6379
Adding replica 10.10.10.107:6380 to 10.10.10.106:6379
Adding replica 10.10.10.104:6380 to 10.10.10.104:6379
M: e59449112f33dcb2dfad7a1ec32920470f589c32 10.10.10.104:6379
   slots:10923-16383 (5461 slots) master
M: 566762510d6b7b2e1b361a8a8d44e4a9d788a92f 10.10.10.106:6379
   slots:5461-10922 (5462 slots) master
M: 8984e7d3e53ddcec5dd59684f69a1976a4db33ef 10.10.10.107:6379
   slots:0-5460 (5461 slots) master
S: 8e222ceb6ad4a9ca48566bd467b0e1b6573c2fc0 10.10.10.104:6380
   replicates e59449112f33dcb2dfad7a1ec32920470f589c32
S: 0c9e17b5955be559a7edf2853bff02d7415ea72f 10.10.10.106:6380
   replicates 8984e7d3e53ddcec5dd59684f69a1976a4db33ef
S: a8e54df5776b412de65b904ac3928d0d308fdc15 10.10.10.107:6380
   replicates 566762510d6b7b2e1b361a8a8d44e4a9d788a92f
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join....
>>> Performing Cluster Check (using node 10.10.10.104:6379)
M: e59449112f33dcb2dfad7a1ec32920470f589c32 10.10.10.104:6379
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
M: 8984e7d3e53ddcec5dd59684f69a1976a4db33ef 10.10.10.107:6379
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: a8e54df5776b412de65b904ac3928d0d308fdc15 10.10.10.107:6380
   slots: (0 slots) slave
   replicates 566762510d6b7b2e1b361a8a8d44e4a9d788a92f
M: 566762510d6b7b2e1b361a8a8d44e4a9d788a92f 10.10.10.106:6379
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: 0c9e17b5955be559a7edf2853bff02d7415ea72f 10.10.10.106:6380
   slots: (0 slots) slave
   replicates 8984e7d3e53ddcec5dd59684f69a1976a4db33ef
S: 8e222ceb6ad4a9ca48566bd467b0e1b6573c2fc0 10.10.10.104:6380
   slots: (0 slots) slave
   replicates e59449112f33dcb2dfad7a1ec32920470f589c32
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
中间需要我们输入一个yes确定主从的分配。通过日志就可以看到master和slave的一个分配情况，以及slot的分配。那个slots是跟存储的时候hash有关的，即一个字符串先要经过哈希，知道他应该存储到那个节点上，然后才会存储到对应的server中。当我们用普通的api去查询的时候，需要查那个真正存储的机器，才能读取到数据。当然使用cluster api就不会有这个问题了。

#### 2.6 集群环境验证

下面我们验证下集群的效果：

首先查看一下集群的状态，然后创建一个key，再换另一个节点登录，看看能否查询到。
```
[root@localnode6 redis-3.2.10]# ./src/redis-cli -h 10.10.10.104 -c -p 6379
10.10.10.104:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_sent:86
cluster_stats_messages_received:86
10.10.10.104:6379> get foo
(nil)
10.10.10.104:6379> set foo bar
OK
10.10.10.104:6379> get foo
"bar"
10.10.10.104:6379>
[root@localnode6 redis-3.2.10]# ./src/redis-cli -h 10.10.10.106 -c -p 6379
10.10.10.106:6379> get foo
-> Redirected to slot [12182] located at 10.10.10.104:6379
"bar"
10.10.10.104:6379>
```
这样我们的集群环境就搭建完了。

万里长征又多走了一步~

## 安装监控软件

如果用windows办公，还是有不少图形化的工具，可以连接redis的，比如`redis desktop`:

首先可以去下面网址去下载对应的版本：
https://redisdesktop.com/download

然后就是无脑安装了，安装完登录到对应的机器上即可。不过貌似是单节点登录，即你看不到集群的数据。可能是我不会用 :-(

![](http://images2017.cnblogs.com/blog/449064/201708/449064-20170809194801183-1304233207.png)


## 使用Jedis API访问Redis Cluster

首先在pom.xml中引入Jedis的jar包：
```
<dependency>
   <groupId>redis.clients</groupId>
   <artifactId>jedis</artifactId>
   <version>2.9.0</version>
</dependency>
```
然后测试一下：
```
public class JedisTest {
    public static void main(String[] args) {
        Jedis redis = new Jedis("10.10.10.104",6379);
        String foo = redis.get("foo");
        System.out.println(foo);
    }
｝
```

## 使用JedisCluster访问Redis Cluster

参考下这片文章就行，试验过了可用的：
http://www.cnblogs.com/shihaiming/p/5953956.html

## 参考

1 安装redis经常遇到的问题：http://www.cnblogs.com/HKUI/p/4439575.html
2 make的时候error: jemalloc/jemalloc.h报错：http://openskill.cn/article/151
3 linux常用目录介绍：http://www.linuxidc.com/Linux/2016-08/134701.htm
4 redis集群环境搭建：http://www.cnblogs.com/wuxl360/p/5920330.html
5 redis安装：http://www.runoob.com/redis/redis-install.html
6 redis官方文档:https://redis.io/download
7 redis client官方文档：https://github.com/uglide/RedisDesktopManager
8 redis desktop windows下载：https://redisdesktop.com/download