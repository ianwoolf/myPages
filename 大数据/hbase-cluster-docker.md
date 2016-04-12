+++
date = "2015-10-20T16:30:09+08:00"
draft = true
title = "【原创】基于docker，一键搭建hbase集群"

+++
# 一键搭建Hbase的docker集群

代码见[github](https://github.com/ianwoolf/DockerFiles/tree/master/hadoop)，不赘述。引用了某位大神的hadoop集群，自己修改后，增加的hbase集群。

我用的hadoop2.6.2，hbase1.2.0，验证ok。

需要注意以下：

- 用start-hbase.sh启动集群后，需要手动ssh一下各slave，完成`yes`输入，否则启动会有问题。
- 如果要启动3台以上的集群，需要手动修改配置文件（hadoop:core-site.xml;yarn-site.xml;slave hbase:slaves;hbase-site.xml;regionservers）
- start-all.sh   start-hbase.sh启动jiqun

### 未解问题
当我docker的host设定成`master.zzz/slave1.zzz/slave2.zzz`，并且hbase、hadoop配置文件也做相应修改后，hbase会无故多出slave1和slave2两个regionserver。原因不明，怀疑和dnsmasq、配置、host有关，具体原因未查。
