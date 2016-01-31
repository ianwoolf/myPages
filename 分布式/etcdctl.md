+++
date = "2015-10-20T16:30:09+08:00"
draft = true
title = "【原创】etcdctl"

+++
### 导语
先介绍对于数据的命令和操作，然后介绍集群的命令及操作
### 数据设置
    $ etcdctl  -h
    NAME:
       etcdctl - A simple command line client for etcd.    

    USAGE:
       etcdctl [global options] command [command options] [arguments...]    

    VERSION:
       2.2.3    

    COMMANDS:
       backup               backup an etcd directory
       cluster-health       check the health of the etcd cluster
       mk                   make a new key with a given value
       mkdir                make a new directory
       rm                   remove a key or a directory
       rmdir                removes the key if it is an empty directory or a key-value pair
       get                  retrieve the value of a key
       ls                   retrieve a directory
       set                  set the value of a key
       setdir               create a new or existing directory
       update               update an existing key with a given value
       updatedir            update an existing directory
       watch                watch a key for changes
       exec-watch           watch a key for changes and exec an executable
       member               member add, remove and list subcommands
       import               import a snapshot to a cluster
       user                 user add, grant and revoke subcommands
       role                 role add, grant and revoke subcommands
       auth                 overall auth controls
       help, h              Shows a list of commands or help for one command
       
    GLOBAL OPTIONS:
       --debug                      output cURL commands which can be used to reproduce the request
       --no-sync                    don't synchronize cluster information before sending request
       --output, -o 'simple'        output response in the given format (`simple`, `extended` or `json`)
       --discovery-srv, -D          domain name to query for SRV records describing cluster endpoints
       --peers, -C                  a comma-delimited list of machine addresses in the cluster (default: "http://127.0.0.1:4001,http://127.0.0.1:2379")
       --endpoint                   a comma-delimited list of machine addresses in the cluster (default: "http://127.0.0.1:4001,http://127.0.0.1:2379")
       --cert-file                  identify HTTPS client using this SSL certificate file
       --key-file                   identify HTTPS client using this SSL key file
       --ca-file                    verify certificates of HTTPS-enabled servers using this CA bundle
       --username, -u               provide username[:password] and prompt if password is not supplied.
       --timeout '1s'               connection timeout per request
       --total-timeout '5s'         timeout for the command execution (except watch)
       --help, -h                   show help
       --version, -v                print the version

### 常用命令
#### 数据命令
    root@6c99219fcb5d:~# etcdctl  ls /
    root@6c99219fcb5d:~# etcdctl mkdir /test
    root@6c99219fcb5d:~# etcdctl ls /
    /test

#### 集群命令
##### 参数
默认会访问`127.0.0.1:2379` 。如果启动时 `-listen-client-urls`没有监听`127.0.0.1`，则使用etcdctl必须加参数 `--peers http://172.17.0.136:2379`  或者环境变量 `ETCDCTL_PEERS=https://127.0.0.1:4001`

#### update
>In this example let's update a8266ecf031671f3 member ID and change its peerURLs value to http://10.0.1.10:2380

    etcdctl member update a8266ecf031671f3 http://10.0.1.10:2380
修改一个  机器其他member会一起改

#### remove
>It is safe to remove the leader, however the cluster will be inactive while a new leader is elected. This duration is normally the period of election timeout plus the voting process

    $ etcdctl member remove c46621e00364182a
    
    Removed member c46621e00364182a from cluster
    $ etcdctl member list
    
    3912a8c78d3e069e: name=etcd1 peerURLs=http://172.17.0.135:2380 clientURLs=http://172.17.0.135:2379
remove的etcd，自动退出
#### add
Adding a member is a two step process:

1. Add the new member to the cluster via the members API or the etcdctl member add command.
2. Start the new member with the new cluster configuration, including a list of the updated members (existing members + the new member).

###### step one

    $ etcdctl member add infra3 http://10.0.1.13:2380
    
    added member 9bf1b35fc7761a23 to cluster
    ETCD_NAME="infra3"
    ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
    ETCD_INITIAL_CLUSTER_STATE=existing
###### step two

    $ export ETCD_NAME="infra3"
    $ export ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
    $ export ETCD_INITIAL_CLUSTER_STATE=existing
    $ etcd -listen-client-urls http://10.0.1.13:2379 \
    -advertise-client-urls http://10.0.1.13:2379  \
    -listen-peer-urls http://10.0.1.13:2380 \
    -initial-advertise-peer-urls http://10.0.1.13:2380 \
    -data-dir %data_dir%

##### 亲测
新建之后，member list查看member号码，删除之再新加
    
    # 先删除
    $ etcdctl member remove c46621e00364182a
    Removed member c46621e00364182a from cluster
    
    # 已删除，cluster中只剩一台
    $ etcdctl member list
    3912a8c78d3e069e: name=etcd1 peerURLs=http://172.17.0.135:2380 clientURLs=http://172.17.0.135:2379

    # 向cluster中新加member
    $ etcdctl member add etcd136 http://172.17.0.136:2380
    Added member named etcd136 with ID bf77c13251c2d245 to cluster

    ETCD_NAME="etcd136"
    ETCD_INITIAL_CLUSTER="etcd1=http://172.17.0.135:2380,etcd136=http://172.17.0.136:2380"
    ETCD_INITIAL_CLUSTER_STATE="existing"
此时已通过etcdctl向集群新加member，然后对应启动集合，不过注意`-initial-cluster-state`这个参数，追加的member需要`existing`启动

这里就不具体列举了。135、136集群启动，136被删除后再追加到集群中，语句汇总如下，注意136的前后变动，只是`-initial-cluster-state`这个参数，前后两种使用`/`分割了。

    135$ etcd -name etcd1 -initial-advertise-peer-urls http://172.17.0.135:2380 -listen-peer-urls http://172.17.0.135:2380 -listen-client-urls http://172.17.0.135:2379,http://127.0.0.1:2379  -advertise-client-urls http://172.17.0.135:2379 -initial-cluster etcd1=http://172.17.0.135:2380,etcd136=http://172.17.0.136:2380 -initial-cluster-state new -data-dir=/data/etcd/
    
    136$ etcd -name etcd136 -initial-advertise-peer-urls http://172.17.0.136:2380 -listen-peer-urls http://172.17.0.136:2380 -listen-client-urls http://172.17.0.136:2379  -advertise-client-urls http://172.17.0.136:2379 -initial-cluster etcd136=http://172.17.0.136:2380,etcd1=http://172.17.0.135:2380 -initial-cluster-state new/existing -data-dir=/data/etcd/
~

其实就是`-initial-cluster-state` new => existing

#### 注意
新加入的member，data-dir要干净

#### error case:
- 新增member启动的时候，没把自己加入到 -initial-cluster
- 启动的时候，没有启动add中指定的address（peerurl）
- data-dir指向了： 之前remove掉的但是没删掉的、仍在在服务中的-data-dir。will exit automatically（etcd: this member has been permanently removed from the cluster. Exiting.）

