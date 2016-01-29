+++
date = "2015-10-20T16:30:09+08:00"
draft = true
title = "【原创】etcd配置"

+++
    [root@localhost ~]# wget https://github.com/coreos/etcd/releases/download/v0.4.6/etcd-v0.4.6-linux-amd64.tar.gz
    [root@localhost ~]# tar -zxvf etcd-v0.4.6-linux-amd64.tar.gz
    [root@localhost ~]# cd etcd-v0.4.6-linux-amd64
    [root@localhost etcd-v0.4.6-linux-amd64]# cp etcd* /bin/
    [root@localhost ~]# etcd -version
    etcd version 0.4.6
#### 简单操作
    letong@me:~$ curl -L http://192.168.0.123:4001/v2/keys/lekey -XPUT -d value=”this is key”  #添加
    {“action”:”set”,”node”:{“key”:”/lekey”,”value”:”this is key”,”modifiedIndex”:4,”createdIndex”:4}}
    letong@me:~$ curl -L http://192.168.0.123:4001/v2/keys/lekey #查询
    {“action”:”get”,”node”:{“key”:”/lekey”,”value”:”this is key”,”modifiedIndex”:4,”createdIndex”:4}}
    letong@me:~$ curl -L http://192.168.0.123:4001/v2/keys/lekey -XDELETE #删除
    {“action”:”delete”,”node”:{“key”:”/lekey”,”modifiedIndex”:5,”createdIndex”:4},”prevNode”:{“key”:”/lekey”,”value”:”this is key”,”modifiedIndex”:4,”createdIndex”:4}}

https://github.com/coreos/etcd/blob/master/Documentation/configuration.md

> offficial etcd port is 2379/2380.  some legacy code and document still references ports 4001, 7001

##### member flag
-name human-readable name for this member default:default  env variable:etcd_name

-data-dir  path to data director. default:${name}.etcd   env: etcd_data_dir

-wal-dir  path to the dedicated wal directory  etcd_wal_dir

-listen-peer-urls:  List of URLs to listen on for peer traffic. This flag tells the etcd to accept incoming requests from its peers on the specified scheme://IP:port combinations. scheme can be either http or https. if 0.0.0.0 is specified as the ip, etcd listens to the given port on all interfaces. multiple urls may be used to specify a number of addresses and ports to liston on. domain name is invalid for biding. default 2380 7001

-listen-client-urls: List of urls to listen on for client traffic. this flag tells the etcd to accept incoming requests from the clients on the specified scheme://IP:port combinations. schema can be http or https. if 0.0.0.0 is specified ad the ip, etcd listen to the given port on all interfaces. default 2370 4001

-max-snapshots

-max-wals

-cors: comma-separated white list of origins for cors

##### clustering flag

-initial: prefix flags are used in bootstraping a new member, and ignored when restaring an existing member

-discovery: prefix flags need to be set when using discovery service

-initial-advertise-peer-urls: List of this members's peer urls to advertise to the rest of the cluster. these addresses are used for communicating etcd data around the cluster. url can contain domain names.  default 2380 7001

-initial-cluster: Initial cluster configuration for bootstrapping. default: http://localhost:2380,default=http://localhost:7001     
                 Note that the URLs specified in initial-cluster are the advertised pee
                 
-initial-cluster-token: default: "etcd-cluster"

-advertise-client-urls: list of this member's client url to advertise to the restu of cluster. default 2379 4001

-discovery.......

##### proxy flag 
##### security flag 
##### Logging Flags
-debug

-log-package-levels
##### Unsafe Flags
##### Experimental Flags
#####  ...
