+++
date = "2015-10-20T16:31:14+08:00"
draft = true
title = "制作带有ssh的docker镜像,转,小修改并亲测"

+++

##  手动搞，然后commit
**个人最爱的方式,我喜欢手动搞出来,然后用dockerfile搞启动命令/环境/网络等**

步骤如下：

首先，使用我们最熟悉的 「-ti」参数来创建一个容器(我之前的小黑是14.10的，所以没用04的，直接用的lastext):`$ sudo docker run -ti ubuntu  /bin/bash`


    root@fc1936ea8ceb:/# sshd
    bash: sshd: command not found
使用 sshd 开启 ssh server 服务,发现没有安装这个服务,注意，我们在使用 「-ti /bin/bash」 进入容器后，获得的是 root 用户的bash


	root@fc1936ea8ceb:/# apt-get install openssh-server
	Reading package lists... Done
	Building dependency tree
	Reading state information... Done
	The following extra packages will be installed:
  	ca-certificates krb5-locales libck-connector0 libedit2 libgssapi-krb5-2
 	 libidn11 libk5crypto3 libkeyutils1 libkrb5-3 libkrb5support0
 	 libpython-stdlib libpython2.7-minimal libpython2.7-stdlib libwrap0 libx11-6
    。。。。
	Do you want to continue? [Y/n] y	
    。。。，，
    Setting up ssh-import-id (3.21-0ubuntu1) ...
	Processing triggers for libc-bin (2.19-0ubuntu6.6) ...
	Processing triggers for ca-certificates (20130906ubuntu2) ...
	Updating certificates in /etc/ssl/certs... 164 added, 0 removed; done.
	Running hooks in /etc/ca-certificates/update.d....done.
	Processing triggers for ureadahead (0.100.0-16) ...
    
* 14.04的ubantu的docker镜像没有sshd，lastest是有的。**如果要用04，还需要先升级下apt-get**


需要建立一个文件夹` mkdir -p /var/run/sshd`


	# /usr/sbin/sshd -D &
	root@fc1936ea8ceb:/# netstat -ntlp
		Active Internet connections (only servers)
		Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
		tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      3238/sshd       
		tcp6       0      0 :::22                   :::*                    LISTEN      3238/sshd   
	# mkdir root/.ssh
	# vi /root/.ssh/authorized_keys  #把中控机或者常用机器的公钥填进去（如非有中控，推荐以后空着）
    # sed -ri 's/session    required     pam_loginuid.so/#session    required     pam_loginuid.so/g' /etc/pam.d/sshd  #修改 ssh 服务的安全登陆配置

    # 启动docker时执行的初始化脚本，这里写的启动ssh。这个应该做成开机启动，这里仅做示例
    # vi /run.sh 
    # chmod +x run.sh
	# cat /run.sh
 	   #!/bin/bash
		/usr/sbin/sshd -D


然后我们退出，保存镜像


    # exit
    $ sudo docker commit  xxxx sshd:ubantu  #xxx为刚才容器的CONTAINER ID
    $ sudo docker  images   # 查看是否有了
		REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
		sshd                ubuntu              7aef2cd95fd0        10 seconds ago      255.2 MB
		ubuntu              latest              ba5877dc9bec        3 months ago        192.7 MB

**我们来验证，启动一个container，连docker主机**


	$ sudo docker  run -p 100:22  -d sshd:ubuntu /run.sh  #启动容器，并映射端口 100 -->22
		3ad7182aa47f9ce670d933f943fdec946ab69742393ab2116bace72db82b4895
	$ sudo docker ps
	#找一台机器，连过去试试

	$ ssh root@xxxx -p 100   #xxxx 是docker主机的ip，100是端口号。  这个跟container启动方式有关
		。。。。
		Are you sure you want to continue connecting (yes/no)? yes
    。。。。
    root@3ad7182aa47f:~#
**成功登陆，镜像创建成功。**

run.sh 脚本内容

## dockerfile搞
首先，创建一个叫做 sshd_ubuntu 的文件夹，用于存放我们的 Dockerfile 、脚本文件、以及其他文件。

	$ mkdir sshd_ubuntu
	$ cd sshd_ubuntu/
	$ touch Dockerfile run.sh
	$ ls
		Dockerfile  run.sh

编写 run.sh(启动sshd，同上) 和 authorized_keys（登陆公钥，根据自己情况选择）`$ cat ~/.ssh/id_rsa.pub >authorized_keys`

**重点来了，下面是 Dockerfile 的内容**

	FROM ubuntu:latest  #设置继承镜像


	MAINTAINER dwj_zz@163.com #作者的信息
	#更改源，根据自己需求，我改的网易的源
	RUN echo "deb http://mirrors.163.com/ubuntu/ trusty main restricted universe multiverse" > /etc/apt/sources.list
	RUN echo "deb http://mirrors.163.com/ubuntu/ trusty-security main restricted universe multiverse" >> /etc/apt/sources.list
	RUN echo "deb http://mirrors.163.com/ubuntu/ trusty-updates main restricted universe multiverse" >> /etc/apt/sources.list
	RUN echo "deb http://mirrors.163.com/ubuntu/ trusty-proposed main restricted universe multiverse" >> /etc/apt/sources.list
	RUN echo "deb http://mirrors.163.com/ubuntu/ trusty-backports main restricted universe multiverse" >> /etc/apt/sources.list
	RUN apt-get update

	#安装 ssh 服务
	RUN apt-get install -y openssh-server
	RUN mkdir -p /var/run/sshd
	RUN mkdir -p /root/.ssh
	#取消pam限制
	RUN sed -ri 's/session    required     pam_loginuid.so/#session    required     pam_loginuid.so/g' /etc/pam.d/sshd

	#复制配置文件到相应位置,并赋予脚本可执行权限
	ADD authorized_keys /root/.ssh/authorized_keys
	ADD run.sh /run.sh
	RUN chmod 755 /run.sh

	#开放端口
	EXPOSE 22

	#设置自启动命令
	CMD ["/run.sh"]

创建镜像

	$ sudo docker build -t sshd:dockerfile .
		Sending build context to Docker daemon 5.632 kB
		Sending build context to Docker daemon
		Step 0 : FROM ubuntu:latest
		 ---> ba5877dc9bec
		Step 1 : MAINTAINER dwj_zz@163.com
		 ---> Running in 188d74d02d35
		 ---> 473eb019b331
		Removing intermediate container 188d74d02d35
		# 使用 Dockerfile 创建，他会帮你删除中间无用的文件层
		Step 2 : RUN echo "deb http://mirrors.163.com/ubuntu/ trusty main restricted universe multiverse" > /etc/apt/sources.list
		 ---> Running in f52e2a583db5
		 ---> bd4ceef2ee19
		Removing intermediate container f52e2a583db5
		Step 3 : RUN echo "deb http://mirrors.163.com/ubuntu/ trusty-security main restricted universe multiverse" >> /etc/apt/sources.list
	......
		Step 15 : EXPOSE 22
		 ---> Running in 660a57c41b25
		 ---> 3ff9e88c4847
		Removing intermediate container 660a57c41b25
		Step 16 : CMD /run.sh
 		---> Running in 5a0d838b6759
 		---> 532f12181013
		Removing intermediate container 5a0d838b6759
		Successfully built 532f12181013
最后，返回告诉我们创建成功，镜像 id 号是532f12181013，让我们来查看下

	$ docker images
	REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL 	SIZE
	sshd                dockerfile          532f12181013        7 seconds ago       242 MB
	sshd                ubantu              08fdb11b77b5        About an hour ago   229.9 MB
最后，测试镜像，运行容器。同上，不在赘述。
**注意到，后面跟上面手动搞镜像启动命令不一样，不需要在输入命令‘/run.sh’了，因为已经在 Dockerfile 中定义了自启动命令。**

