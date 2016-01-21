+++
date = "2016-01-21T22:31:14+08:00"
draft = true
title = "zooker镜像制作，集群搭建"

+++
###  zookeeper镜像制作及启动
#### 先java
zookeeper需要java环境，所以首先制作一个java镜像。这里省略步骤，采用之前制作的ubuntu的sshd镜像，添加如下语句，按照java环境

    sudo apt-get install default-jdk

#### zookeeper
然后制作zookeeper镜像

    FROM java:my
    MAINTAINER ian woolf <btw515wolf2@gmail.com> 

    ADD http://ftp.nluug.nl/internet/apache/zookeeper/stable/zookeeper-3.4.6.tar.gz /zookeeper.tar.gz

    RUN mkdir -p zoo && \
      cd zoo && \
      tar xvfz ../zookeeper.tar.gz --strip-components=1 && \
      rm /zookeeper.tar.gz

    ADD start.sh /start.sh
    RUN chmod 755 /start.sh
    EXPOSE 22 2888 3888 2181

    CMD ["/start.sh"]
start.sh 文件

    echo "root" | passwd –stdin root
    echo "root:root"|chpasswd
    sed -i '/^PermitRootLogin/s/without-password/yes/g' /etc/ssh/sshd_config
    /etc/init.d/ssh restart

    cp /zoo/conf/zoo_sample.cfg /zoo/conf/zoo.cfg
    mkdir -p /var/lib/zookeeper
    /zoo/bin/zkServer.sh start-foreground

    /bin/bash
#### 集群启动
一键启动集群的方法其实很简单。把data和conf目录挂出来，先生成好conf和myid，启动docker直接启动就尅

p.s: `docker inspect --format '{{ .NetworkSettings.IPAddress }}'`查看ip

### 安装及使用
- 编写配置 增加server及端口信息。 必须server.no:port1:port2
第一个端口（ port ）是从（ follower ）机器连接到主（ leader ）机器的端口，第二个端口是用来进行 leader 选举的端口
- 在data目录添加myid，server的no
- zkServer.sh start  status看状态
#### state结构
```
czxid. 节点创建时的zxid.
mzxid. 节点最新一次更新发生时的zxid.
ctime. 节点创建时的时间戳.
mtime. 节点最新一次更新发生时的时间戳.
dataVersion. 节点数据的更新次数.
cversion. 其子节点的更新次数.
aclVersion. 节点ACL(授权信息)的更新次数.
ephemeralOwner. 如果该节点为ephemeral节点, ephemeralOwner值表示与该节点绑定的session id. 如果该节点不是ephemeral节点, ephemeralOwner值为0. 至于什么是ephemeral节点, 请看后面的讲述.
dataLength. 节点数据的字节数.
numChildren. 子节点个数.
```
#### 命令、参数及注意事项
```
create -e 临时节点  -s 顺序节点
ls / true 表明监听子节点节点增删、及本身删除   
get / true 表明监听节点更新和删除   对一个getW
stat / true 获取znode状态 表明监听node更新和删除 
watch触发一次之后，会从列表中删除，需要重新watch
```
### 文档
 Zookeeper不是用来做数据库或者存贮大对象的。相反，它只负责协调数据。数据可以来自配置表单、结构化信息等等。这些数据的有一个共同的特点那就是都很小：以Kb为测量单位。Zookeeper的client和server的实现类都会验证znode存储的数据是否小于1M，但是数据应该比平均值小的多。操作大数据将会触发一些消耗时间的额外操作并且影响潜在的操作，因为需要额外的时间在网络和存储介质上转移数据。如果有大数据需要存储，通常的办法是把这些数据存储在专门的大型文件系统上，例如NFS或者HDFS，然后把存储介质的位置存在zookeeper上。

Ephemeral Nodes
ZooKeeper also has the notion of ephemeral nodes. These znodes exists as long as the session that created the znode is active. When the session ends the znode is deleted. Because of this behavior ephemeral znodes are not allowed to have children.

Sequence Nodes -- Unique Naming
When creating a znode you can also request that ZooKeeper append a monotonically increasing counter to the end of path. This counter is unique to the parent znode. The counter has a format of %010d -- that is 10 digits with 0 (zero) padding (the counter is formatted in this way to simplify sorting), i.e. "<path>0000000001". See Queue Recipe for an example use of this feature. Note: the counter used to store the next sequence number is a signed int (4bytes) maintained by the parent node, the counter will overflow when incremented beyond 2147483647 (resulting in a name "<path>-2147483647").

### 配置文件等例子
#### 配置文件 conf/zoo.conf

    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/tmp/zookeeper
    clientPort=2181
    #maxClientCnxns=60
    #autopurge.snapRetainCount=3
    #autopurge.purgeInterval=1
    server.1=172.17.0.121:2888:3888
    server.2=172.17.0.122:2888:3888
#### myid文件
/tmp/zookeeper/myid

#### 特别注意
配置文件中，必须为server.x， myid中为对应的x

### golang测试代码，todo

    "go.intra.xiaojukeji.com/golang/go-zookeeper/zk"

    type ZH struct {
        conn *zk.conn
    }
    (z *ZH)
    data,_,err := z.conn.Get(path)
    data, _, event, err := z.conn.GetW(path)
    z.conn.Create(path, data, flags, ZK_ACL)
    
    //verison冲突、有子node无法删除
    conn.Delete(path, version)

