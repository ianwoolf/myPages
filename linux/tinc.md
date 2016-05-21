+++
date = "2016-02-15T16:30:09+08:00"
draft = true
title = "【原创】tinc在各种平台上的搭建及配置"

+++
## tinc
先抄一段
>we will go over how to use Tinc, an open source Virtual Private Network (VPN) daemon, to create a secure VPN that your servers can communicate on as if they were on a local network. We will also demonstrate how to use Tinc to set up a secure tunnel into a private network. 

>A few of the features that Tinc has that makes it useful include encryption, optional compression, automatic mesh routing (VPN traffic is routed directly between the communicating servers, if possible), and easy expansion. These features differentiate Tinc from other VPN solutions such as OpenVPN, and make it a good solution for creating a VPN out of many small networks that are geographically distributed. Tinc is supported on many operating systems, including Linux, Windows, and Mac OS X.

我的初衷是我要用我的树莓派来远程监控和控制我家里的树莓派。所以我搭建了一个vpn，用vps做server，这样就可以随时随地的访问和控制我的树莓派，比如看看家里的照片等，和所有能连到树莓派上的智能家居。然后还有一个用处，是我可以随时随地访问公司里的电脑，不需要把电脑拿回家（在公司用我自己的mac，不会弄远程访问，也舍不得天天把mac放在公司）。
## tincd命令
    sudo tincd -c ./zzzvpn/ --pidfile ./zzzvpn/tinc.pid --logfile ./zzzvpn/tinc.log -d=INFO -D
自行`tincd --help`查看
## 以linux为例，配置和启动
按照tinc之后，linux在/etc/tinc里面。然后开始进行server/client的配置。

每个tinc配置，需要有`tinc.conf`，`hosts文件夹`，`tinc-up`，`tinc-down`

我们先配置server端，假设我们在`/etc/tinc/zzzvpn`下，我们的vpn叫`zzzvpn`
### server端配置
#### tinc.conf 
    Name=zzzs
    Interface=zzzvpn
    AddressFamily = ipv4
    Port=756
    PrivateKeyFile=/etc/tinc/zzzvpn/rsa_key.priv
    
#### tinc-up(linux   os  都一样)
    ifconfig $INTERFACE 10.3.0.1 netmask 255.255.255.0
#### tinc-down(linux   os  都一样)
    ifconfig $INTERFACE down
#### hosts/zzzs
    Compression=9
    Address = your vps ip
    Subnet = 10.3.0.1/32
    Port=xxx
#### 生成公私钥
     tincd -n zzzvpn -K
 可以看见生成了/etc/tinc/zzzvpn/rsa_key.priv和hosts下zzzvpn的私钥。我们稍等在启动，先配置client。
### client配置
#### tinc.conf
    ConnectTo=zzzs
    Port=756
    Name=vultr
    AddressFamily=ipv4
    Interface=zzzvpn
    PrivateKeyFile=/etc/tinc/zzzvpn/rsa_key.priv
#### tinc-up tinc-down一样，注意up中的内网ip
#### hosts/vultr
    Compression=9
    Subnet=10.3.0.2/32
#### 生成公私钥
    tincd -n zzzvpn -K。
### 拷贝hosts
然后把vul上的hosts/vultr  和 zzzs上的hosts/zzzs互相拷贝到对方的hosts下。
### 启动
在server和client执行`tincd -n zzzvpn`启动tincd，ping内网ip测试，如果ping通则说明vpn已经ok了。

## 树莓派
和linux一样，需要注意两个地方
1. 命令带sudo
2. 先配置树莓派网络，可访问外网之后。升级apt

    sudo apt-get update
    sudo apt-get install tinc

其他和linux一样，内网ip不重复就可以。
## mac还没通，先记录
    brew install tuntap
    sudo kextload /Library/Extensions/tap.kext
    sudo kextload /Library/Extensions/tun.kext


## macOS上配置tinc
这个测试了很久，闲话不说，上东西


tinc.conf

    ConnectTo = zzzs
    Port=756
    Name = mac
    PrivateKeyFile = /usr/local/etc/tinc/xxx/rsa_key.priv
    Device = /dev/tap0
    
tinc-up

    ifconfig $INTERFACE 10.3.0.3 netmask 255.255.255.0
    
tinc-down

    ifconfig $INTERFACE down
    
host/client

    Subnet=10.3.0.3/32
    Compression=9

    -----BEGIN RSA PUBLIC KEY-----
    -----END RSA PUBLIC KEY-----
命令

    停止
    for pid in `ps aux|grep tinc|grep -v grep|awk '{print $2}'\`;do sudo kill -9 $pid ;done
    启动
    sudo tincd -c /usr/local/etc/tinc/zzzvpn -D --debug --pidfile=./tinc.pid --logfile=./tinc.log
    
注意：  

- Device
- 配置完成后，`tincd -n zzzvpn -K` 生成公私钥
- 生成的host，要拷给server
- 启动后，可以直接ssh到集群任意机器
