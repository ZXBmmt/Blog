## 概述

ZooKeeper是一种用于分布式应用程序的分布式开源协调服务。 它公开了一组简单的原语，分布式应用程序可以构建这些原语，以实现更高级别的服务，以实现同步，配置维护以及组和命名。 它被设计为易于编程，并使用在熟悉的文件系统目录树结构之后设计的数据模型。 它在Java中运行，并且具有Java和C的绑定。
众所周知，协调服务很难做到。 他们特别容易出现比赛条件和死锁等错误。 ZooKeeper背后的动机是减轻分布式应用程序从头开始实施协调服务的责任。

## 下载
```bash
$ cd /data
$ wget http://mirrors.shu.edu.cn/apache/zookeeper/stable/zookeeper-3.4.13.tar.gz
```
## 快速体验
```bash
$ tar -zxvf zookeeper-3.4.13.tar.gz
$ cd zookeeper-3.4.13/conf
#初始化配置文件
$ cp zoo_sample.cfg zoo.cfg
$ cd ../bin
#启动服务
$ zkServer.sh start
#客户端连接服务器
$ zkCli.sh -server 127.0.0.1:2181
#帮忙 可查看基本命令
[zkshell: 0] help
ZooKeeper host:port cmd args
    get path [watch]
    ls path [watch]
    set path data [version]
    delquota [-n|-b] path
    quit
    printwatches on|off
    createpath data acl
    stat path [watch]
    listquota path
    history
    setAcl path acl
    getAcl path
    sync path
    redo cmdno
    addauth scheme auth
    delete path [version]
    setquota -n|-b val path
#查看节点
[zkshell: 8] ls /
[zookeeper]
#创建节点
[zkshell: 9] create /zk_test my_data
Created /zk_test
[zkshell: 11] ls /
[zookeeper, zk_test]
[zkshell: 12] get /zk_test
my_data
cZxid = 5
ctime = Fri Jun 05 13:57:06 PDT 2009
mZxid = 5
mtime = Fri Jun 05 13:57:06 PDT 2009
pZxid = 5
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0
dataLength = 7
numChildren = 0
```
## 伪集群搭建
生产环境中为了保证服务高可用，zookeeper也应该搭建集群，下面是单机部署多服务实例做伪集群示例
```bash
$ cd /data/zookeeper-3.4.13/bin
$ zkServer.sh stop
$ cd /data
$ mkdir zookeeper
$ cp -rf zookeeper-3.4.13 zookeeper
$ cd zookeeper
$ mv zookeeper-3.4.13/ zookeeper-1
$ cp -rf zookeeper-1/ zookeeper-2
$ cp -rf zookeeper-1/ zookeeper-2
######配置开始
$ cd zookeeper-1/
$ mkdir data
$ mkdir log
#编辑配置文件内容如下
# tickTime=2000
# initLimit=10
# syncLimit=5
# dataDir=/data/zookeeper/zookeeper-1/data
# dataLogDir=/data/zookeeper/zookeeper-1/log
# clientPort=2181
# server.1=127.0.0.1:2888:3888
# server.2=127.0.0.1:2889:3889
# server.3=127.0.0.1:2890:3890
$ vi conf/zoo.cfg
$ cd data/
# 配置myid输入1
$ vi myid
########配置结束

# zookeeper-2 zookeeper3 重复上述配置过程
# 进入各节点bin目录 执行./zkServer.sh start 启动节点
# 所有节点启动后 可执行./zkServer.sh status检查节点状态
```
## JAVA API连接集群
maven pom 依赖
```xml
<dependencies>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.13</version>
    </dependency>
    <dependency>
        <groupId>com.github.sgroschupf</groupId>
        <artifactId>zkclient</artifactId>
        <version>0.1</version>
    </dependency>
</dependencies>
```
测试代码
```java
import org.I0Itec.zkclient.IZkChildListener;
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;

import java.util.List;

public class ZkUseSample {
    private static String zkCluster = "127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183";
    private static ZkClient zkClient = new ZkClient(zkCluster,5000);

    public static void main(String[] args){
        List<String> roots = zkClient.getChildren("/");
        for (String root : roots) {
            System.out.println(root);
        }
        zkClient.deleteRecursive("/mmt");
        zkClient.createPersistent("/mmt","zxb");
        //监听节点  subscribeChildChanges 监听当前节点以及子节点增加或者删除
        zkClient.subscribeChildChanges("/mmt", new IZkChildListener() {
            public void handleChildChange(String s, List<String> list){
                System.out.println("child change路径："+s);
                System.out.println("child change变更的节点为:"+list);
            }
        });
        //监听节点  subscribeDataChanges 监听当前节点内容的变更
        zkClient.subscribeDataChanges("/mmt", new IZkDataListener() {
            public void handleDataChange(String s, Object o){
                System.out.println("data change路径："+s);
                System.out.println("data change变更的内容为:"+o.toString());
            }

            public void handleDataDeleted(String s){
                System.out.println("data deleted路径："+s);
            }
        });
        zkClient.createPersistent("/mmt/test1","test1");
        zkClient.createPersistent("/mmt/test2","test2");
        zkClient.createPersistent("/mmt/test3","test3");
        System.out.println(zkClient.readData("/mmt"));
        System.out.println(zkClient.readData("/mmt/test1"));
        System.out.println(zkClient.readData("/mmt/test2"));
        System.out.println(zkClient.readData("/mmt/test3"));
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        zkClient.writeData("/mmt","mmt11");
        zkClient.writeData("/mmt/test1","test11");
        zkClient.writeData("/mmt/test2","test22");
        zkClient.writeData("/mmt/test3","test33");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        zkClient.delete("/mmt/test3");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        zkClient.deleteRecursive("/mmt");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
```
## zookeeper可视化客户端小工具
* [下载地址](wget https://issues.apache.org/jira/secure/attachment/12436620/ZooInspector.zip)
* unzip解压后 进入到build目录执行java -jar zookeeper-dev-ZooInspector.jar

## zookeeper可视化web端小工具
* https://github.com/zhitom/zkweb
