+++
date = "2016-02-01T16:30:09+08:00"
draft = true
title = "【原创】etcd-api"

+++
Tutorial Series. This tutorial is part 4 of 9 in the series:[Getting Started with CoreOS](https://www.digitalocean.com/community/tutorials/how-to-use-etcdctl-and-etcd-coreos-s-distributed-key-value-store)


####  get a listing of the top-level keys/directories
     $ curl http://172.17.0.135:2379/v2/keys/test

    {"action":"get","node":{"key":"/test","dir":true,"nodes":[{"key":"/test/data","value":"content4","modifiedIndex":12,"createdIndex":11}],"modifiedIndex":8,"createdIndex":8}}

>To modify the behavior of these operations, you can pass in flags at the end of your request using the ?flag=value syntax. Multiple flags can be separated by a & character.

    $ curl http://172.17.0.135:2379/v2/keys/?recursive=true
    {"action":"get","node":{"dir":true,"nodes":[{"key":"/test","dir":true,"nodes":[{"key":"/test/data","value":"content4","modifiedIndex":12,"createdIndex":11}],"modifiedIndex":8,"createdIndex":8},{"key":"/test2","dir":true,"nodes":[{"key":"/test2/1","value":"1","modifiedIndex":29,"createdIndex":29},{"key":"/test2/r1","dir":true,"nodes":[{"key":"/test2/r1/1","value":"1","modifiedIndex":31,"createdIndex":31}],"modifiedIndex":30,"createdIndex":30}],"modifiedIndex":28,"createdIndex":28}]}}

    $ curl -X PUT http://172.17.0.135:2379/v2/keys/test2/1  -d value=curl   # updateorcreate
    {"action":"set","node":{"key":"/test2/1","value":"curl","modifiedIndex":32,"createdIndex":32},"prevNode":{"key":"/test2/1","value":"1","modifiedIndex":29,"createdIndex":29}}

    $ curl -X DELETE http://172.17.0.135:2379/v2/keys/test2/2    
    {"action":"delete","node":{"key":"/test2/2","modifiedIndex":34,"createdIndex":33},"prevNode":{"key":"/test2/2","value":"curl","modifiedIndex":33,"createdIndex":33}}
#### 状态命令
    $ curl -L http://172.17.0.135:2379/version
    {"etcdserver":"2.2.3","etcdcluster":"2.2.0"}
    $ curl -L http://172.17.0.135:2379/v2/stats/leader
    {"leader":"286b439b1ac07ed5","followers":{"a3bba712489b1451":{"latency":{"current":0.002655,"average":0.00537593862434881,"standardDeviation":0.009184038615071406,"minimum":0.002195,"maximum":0.545599},"counts":{"fail":0,"success":104227}}}}[root@li713-250
    $ curl -L http://172.17.0.135:2379/v2/stats/self # name  id  starttime  leaderinfo ....
    {"name":"etcd1","id":"286b439b1ac07ed5","state":"StateLeader","startTime":"2016-02-01T03:04:25.067101704Z","leaderInfo":{"leader":"286b439b1ac07ed5","uptime":"5h17m31.484033141s","startTime":"2016-02-01T03:04:43.970697888Z"},"recvAppendRequestCnt":0,"sendAppendRequestCnt":104413,"sendPkgRate":5.555567792156581,"sendBandwidthRate":400.22310374696013}
   $ curl -L http://172.17.0.135:2379/v2/stats/store # getsuccess  getfail .....
    {"getsSuccess":25,"getsFail":8,"setsSuccess":7,"setsFail":1,"deleteSuccess":8,"deleteFail":5,"updateSuccess":1,"updateFail":0,"createSuccess":15,"createFail":1,"compareAndSwapSuccess":0,"compareAndSwapFail":0,"compareAndDeleteSuccess":0,"compareAndDeleteFail":0,"expireCount":0,"watchers":0}

