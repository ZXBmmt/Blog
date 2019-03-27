# 概述

redis3.0及以上版本支持集群部署，下面会介绍集群部署步骤和java连接方式，以及一些运维操作。

### redis安装
```bash
#安装
$ wget http://download.redis.io/releases/redis-4.0.2.tar.gz
$ tar xzf redis-4.0.2.tar.gz
$ cd redis-4.0.2
$ make
#启动server
$ src/redis-server
#连接到redis服务端
$ src/redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
```
### 伪集群搭建
```bash
$ mkdir cluster-test
$ cd cluster-test/
$ mkdir 7000 7001 7002 7003 7004 7005
#创建redis.conf配置文件，并配置如下内容
#port 7000
#cluster-enabled yes
#cluster-config-file nodes.conf
#cluster-node-timeout 5000
#appendonly yes
$ cd 7000
$ vi redis.conf
## 每个不同端口目录下重复上面建redis.conf文件操作 注意每个端口下的配置文件port参数要和目录一致
#在端口目录下启动redis实例
$ ~/tools/redis-4.0.2/src/redis-server redis.conf &
#重复启动6个

#安装redis ruby库
$ sudo gem install redis -v 3.3.5

#做集群操作
# --replicas 1 为每个主节点做一个从节点
$ ~/tools/redis-4.0.2/src/redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005

#检查集群
$ ~/tools/redis-4.0.2/src/redis-trib.rb check 127.0.0.1:7000

#连接客户端测试集群
$ ~/tools/redis-4.0.2/src/redis-cli -c -h 127.0.0.1 -p 7004
127.0.0.1:7004> get 1
"1-1"
127.0.0.1:7004> set 2 2-2
OK
127.0.0.1:7004> get 2
"2-2"
127.0.0.1:7004> set 1000 aaa
-> Redirected to slot [11326] located at 127.0.0.1:7002
OK
127.0.0.1:7002>
```
### java连接方式

* [项目示例源码传送门](https://github.com/ZXBmmt/redis-cluster)
* 使用JedisCluster

```java
package com.mmt;

import redis.clients.jedis.HostAndPort;
import redis.clients.jedis.JedisCluster;

import java.util.HashSet;
import java.util.Set;

public class JedisClusterSample {

    public static void main(String[] args){
        Set<HostAndPort> jedisClusterNodes = new HashSet<HostAndPort>();
        jedisClusterNodes.add(new HostAndPort("127.0.0.1", 7000));
        jedisClusterNodes.add(new HostAndPort("127.0.0.1", 7001));
        jedisClusterNodes.add(new HostAndPort("127.0.0.1", 7002));
        jedisClusterNodes.add(new HostAndPort("127.0.0.1", 7003));
        jedisClusterNodes.add(new HostAndPort("127.0.0.1", 7004));
        jedisClusterNodes.add(new HostAndPort("127.0.0.1", 7005));
        JedisCluster jc = new JedisCluster(jedisClusterNodes);
        for(int i=0;i<100000;i++){
            String key= i+"";
            try{
              jc.set(key, key);
              String value = jc.get(key);
              System.out.println(value);
            }catch(Exception e){
              e.printStackTrace();
            }
        }
    }
}

```

* 使用Spring Data Redis

```xml
<!-- pom依赖 -->
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
</dependencies>
```
```properties
#application.properties
spring.redis.database=0
spring.redis.jedis.pool.max-active=8
spring.redis.jedis.pool.max-wait=-1
spring.redis.jedis.pool.max-idle=8
spring.redis.jedis.pool.min-idle=0
spring.redis.timeout=500
spring.redis.cluster.max-redirects=6
spring.redis.cluster.nodes=127.0.0.1:7000,127.0.0.1:7001,127.0.0.1:7002,127.0.0.1:7003,127.0.0.1:7004,127.0.0.1:7005
```
```java
package com.mmt.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.List;
/**
*reids集群配置信息
**/
@Component
@ConfigurationProperties(prefix = "spring.redis.cluster")
public class ClusterConfigurationProperties {
    private List<String> nodes;
    private int maxRedirects;

    public List<String> getNodes() {
        return nodes;
    }

    public void setNodes(List<String> nodes) {
        this.nodes = nodes;
    }

    public int getMaxRedirects() {
        return maxRedirects;
    }

    public void setMaxRedirects(int maxRedirects) {
        this.maxRedirects = maxRedirects;
    }
}
```
``` java
package com.mmt.context;

import com.mmt.config.ClusterConfigurationProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisClusterConfiguration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
import org.springframework.data.redis.core.StringRedisTemplate;

/**
*spring reids集群客户端初始化
**/
@Configuration
public class RedisClusterContext {
    @Autowired
    private ClusterConfigurationProperties clusterConfigurationProperties;
    @Bean
    public RedisClusterConfiguration getClusterConfig() {
        RedisClusterConfiguration rcc = new RedisClusterConfiguration(clusterConfigurationProperties.getNodes());
        rcc.setMaxRedirects(clusterConfigurationProperties.getMaxRedirects());
        return rcc;
    }
    @Bean
    public JedisConnectionFactory getConnectionFactory(RedisClusterConfiguration cluster) {
        return new JedisConnectionFactory(cluster);
    }
    @Bean
    public StringRedisTemplate getStringRedisTemplate(RedisConnectionFactory factory) {
        StringRedisTemplate stringTemplate = new StringRedisTemplate();
        stringTemplate.setConnectionFactory(factory);
        return stringTemplate;
    }
}
```
```java
package com.mmt;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.data.redis.core.RedisTemplate;

/**
*测试
**/
@SpringBootApplication
public class SpringbootRedisClusterSample {
    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Bean
    public Object test(){
        for(int i=0;i<100000;i++){
            String key= i+"";
            try {
                redisTemplate.opsForValue().set(key, key + "-" + key);
                String value = redisTemplate.opsForValue().get(key);
                System.out.println(value);
            }catch (Exception e){
                e.printStackTrace();
            }
            try {
                Thread.sleep(1000);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
       return null;
    }
    public static void main(String[] args){
        SpringApplication.run(SpringbootRedisClusterSample.class, args).close();
    }
}
```

### 故障处理
#### 集群重新分片

  ```bash
  ## 将除7000节点外100个hash槽迁移到7000节点
  zxb$ ./redis-trib.rb reshard 127.0.0.1:7000

  How many slots do you want to move (from 1 to 16384)? 100

  Source node #1:all

  ```
  ``` bash
  #迁移前的结果
  M: aca0f8429f137ef379ff152bb9cf220c13cd1527 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
  M: 0984c6d4a3cbee1e5bf4b1be7d9b4362761aabe3 127.0.0.1:7002
      slots:10923-16383 (5461 slots) master
  M: a49cc2414eb23edc53791ba0ececf76da1060a08 127.0.0.1:7004
        slots:5461-10922 (5462 slots) master
  #迁移后的结果
  M: aca0f8429f137ef379ff152bb9cf220c13cd1527 127.0.0.1:7000
   slots:0-5511,10923-10971 (5561 slots) master
  M: 0984c6d4a3cbee1e5bf4b1be7d9b4362761aabe3 127.0.0.1:7002
   slots:10972-16383 (5412 slots) master
  M: a49cc2414eb23edc53791ba0ececf76da1060a08 127.0.0.1:7004
   slots:5512-10922 (5411 slots) master
  ```
  * 问题:我的测试redis版本是4.0.2 所以在使用ruby gem安装的redis库，版本不能使用最新的4以上版本，否则redis-trib.rb reshard 192.168.2.106:8002 重新分片时会报错误(Moving slot 5461 from 127.0.0.1:7004 to 127.0.0.1:7000:
[ERR] Calling MIGRATE: ERR Syntax error, try CLIENT (LIST | KILL | GETNAME | SETNAME | PAUSE | REPLY))。
  * 解决办法:1.卸载最新redis库，gem uninstall redis;2.安装3.x版本，gem install redis -v 3.3.5 测试3.2.1到3.3.5都可以，4.x以上的分片报错

#### 扩容
  
#### 缩减实例
