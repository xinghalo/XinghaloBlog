---
title: Spark Redis的使用
tags: redis
top: 0
---

---
利用Spark来读取Redis的缓存数据
<!-- more -->


# 添加pom引用

首先在pom.xml中添加引用，版本1.0.0是升级spark后自定义的版本。jar包已经发布到maven私库，可以正常使用。

```
<dependency>
    <groupId>com.redislabs</groupId>
    <artifactId>spark-redis</artifactId>
    <version>1.0.0</version>
</dependency>
```

# 在Spark中的应用

在Spark中使用需要先引入redis相关的方法，再基于sc调用各种读写api。

```
package redis

import org.apache.spark.{SparkConf, SparkContext}
import com.redislabs.provider.redis._   //<--这里很重要

object RedisTest {
  def main(args: Array[String]) {
    val sparkConf = new SparkConf()
      .setMaster("local[*]")
      .setAppName("redis-test")
      .set("redis.host", "10.10.5.104")  // 随便配置一台redis主机地址
      .set("redis.port", "6379")         // redis节点对应的端口号

    val sc = new SparkContext(sparkConf)
    sc.setLogLevel("WARN")

    // 例子1 读取前缀为foo的key，返回key列表
    val keysRDD = sc.fromRedisKeyPattern("foo*", 1) //使用相应的API读取数据即可
    keysRDD.collect()
    
    // 例子2 读取名叫foo的kv返回对应的key和value
    val kvRDD = sc.fromRedisKV("foo",1)
    kvRDD.collect()
    
    sc.stop()
  }
}
```