+++
date = "2016-04-17T16:30:09+08:00"
draft = true
title = "【原创】kafka集群搭建"

+++
# kafka集群搭建

## 准备工作

先去[官网](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.9.0.1/kafka_2.10-0.9.0.1.tgz)下载 kafka_2.10-0.9.0.1.tgz 并解压再进入到安装目录（也可以自己配置路径，方法跟配置java、hadoop等路径是一样的）.
    tar -xzf kafka_2.9.2-0.8.1.1.tgz 
    cd kafka_2.9.2-0.8.1.1


我用的ndsmasq做信任关系，不过没有信任关系也没问题（ps,我用hbase的3个容器搭的。后续整理成单独docker)

1.  关闭各台机器的防火墙`/ect/init.d/iptables stop`

2.  进入到打开/ect下的hosts文件,配置host对应ip（我用dnsmasq搞定）
    
        127.0.0.1 localhost
        10.61.5.66 host1
        10.61.5.67 host2
        10.61.5.68 host3
        （ip和机器名根据个人实际情况修改）

## 配置及启动
### 配置zeekeeper
如果使用kafka自带zooker，需要配置zk

进入到kafka安装目录下，zk配置文件为`conf/zookeeper.properties`

    // zk目录，下面需要配置myid
    dataDir=****
注释掉maxClientCnxns=0，在文件末尾添加如下语句

    tickTime=2000
    initLimit=5
    syncLimit=2
    #host1、2、3为主机名，可以根据实际情况更改，端口号也可以更改
    #注意host和id的对应关系，需要配置myid
    server.1=host1:2888:3888
    server.2=host2:2888:3888
    server.3=host3:2888:3888

分别在每台机器上，设置zk对应的id号到zk的dataDir（上面配置的那个目录）`echo 1 >myid`

### 启动zk
启动每台机子的zeekeeper
nohup bin/zookeeper-server-start.sh config/zookeeper.properties >>zk.log &


### 配置kafka
在kafka安装目录下, config目录下配置`server.properties`文件

    zookeeper.connect=host1:2181,host2:2181,host3:2181(zk机器机器端口号根据实际情况修改）

***修改broker.id，注意集群不能有重复！如果已经启动过，请修改配置文件中meta.dir下的id号，或者清空***

producer.properties

    metadata.broker.list=host1:9092,host2:9092,host3:9092(注意端口号和机器)
    prodeucer.type=async

consumer.properties

    zeekeeper.connect=host1:2181,host2:2181,host3:2181(zk集群根据实际情况修改)

### 启动kafka
` nohup bin/kafka-server-start.sh config/server.properties >> kafka.log &

 

## 校验
`bin/kafka-topics.sh --create --zookeeper master:2181 --replication-factor 3 --partitions 6 --topic my-replicated-test`(factor大小不能超过broker数)

查看主题
`bin/kafka-topics.sh --list --zookeeper host1:2181 （也可以是host2:2181等)
my-replicated-test`

查看主题详情
`bin/kafka-topics.sh --describe --zookeeper host1:2181 --topic my-replicated-test`

发送消息
在其中一台机器上建立生产者角色，并发送消息（其实可以是三台机子中的任何一台）

    $ bin/kafka-console-producer.sh --broker-list host1:9092 --topic my-replicated-test
    This is a message
    This is another message

在另一台机器上建立消费者角色（在该终端窗口内可以看到生产者发布这消息）

    bin/kafka-console-consumer.sh --zookeeper host1:2181 --topic my-replicated-test --from- beginning
    This is a message
    This is another message

至此，一个kafka集群就搭好了，可以作为kafka服务器了

## 遇到问题

- 未修改broker.id，id重复导致启动失败。修改后，需要在修改meta.log下的id（我直接清空了）。然后再启动

###### 记录端口情况，以后续写dockerfile
tcp6       0      0 :::9090                 :::*                    LISTEN      17576/java      
tcp6       0      0 :::9092                 :::*                    LISTEN      25105/java      
tcp6       0      0 :::2181                 :::*                    LISTEN      25063/java      
tcp6       0      0 :::42887                :::*                    LISTEN      25105/java      
tcp6       0      0 :::9095                 :::*                    LISTEN      17576/java      
tcp6       0      0 :::7946                 :::*                    LISTEN      52/serf         
tcp6       0      0 172.17.0.17:3788        :::*                    LISTEN      25063/java      
tcp6       0      0 :::7373                 :::*                    LISTEN      52/serf         
tcp6       0      0 :::47632                :::*                    LISTEN      25063/java      
