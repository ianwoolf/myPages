+++
date = "2015-10-20T16:31:14+08:00"
draft = true
title = "用docker起golang的gin服务"

+++
### 首先启动docker 
看到一个可耐的鲸鱼，下面有docker地址，如下语句

    docker is configured to use the default machine with IP 192.168.99.100
    For help getting started, check out the docs at https://docs.docker.com

注意里面的ip，这是docker主机的ip，后面连接container的时候需要通过docker主机的端口转发过去

下面是查找、下载、导出镜像用到的语句

    docker search golang   查找镜像
    docker  pull golang:1.4.2
    docker save -o go-1.4.img golang:1.4.2   导出镜像

### 开始一个container
    docker run -it -p 10101:8080 -v `pwd`/code/go/src/:/go/src/:rw golang:1.4.2 go run /go/src/myLab/golang/gin/main.go

	* -p进行端口转发，docker主机10101 转发到container的8080（gin server端口）
	* -v进行挂载（绝对路径），机器的·pwd·/code/go/src,挂在的container的/go/src。注意依赖
	* 加-d后台运行

Docker ip is 10101   这个端口被转发到container的8080

这样即可跑起来，可curl请求docker主机端口，转发到container进行访问

    curl 192.168.99.100:10101/room/
