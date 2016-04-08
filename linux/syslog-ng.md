+++
date = "2016-04-08T16:30:09+08:00"
draft = true
title = "syslog-ng介绍"

+++
## syslog-ng安装

**centos**：

	wget http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
	rpm -ivh epel-release-6-8.noarch.rpm
	yum --enablerepo=epel install syslog-ng eventlog syslog-ng-libdb
	service syslog-ng start

**ubuntu**（[syslog-ng官方文档](https://syslog-ng.gitbooks.io/getting-started/content/chapters/chapter_0/section_1.html)）：


get release key
    `wget -qO -  http://download.opensuse.org/repositories/home:/laszlo_budai:/syslog-ng/${os}/Release.key | sudo apt-key add -`

add repo to APT sources, `vi /etc/apt/sources.list.d/syslog-ng-obs.list`, add
    `deb  http://download.opensuse.org/repositories/home:/laszlo_budai:/syslog-ng/${os} .`
and

    apt-get update
    apt-get install syslog-ng-core

You can replace Debian_8.0 to any of the supported systems. For Ubuntus there is a 'x' prefix, so the possible values are:

    Debian_7.0
    Debian_8.0
    xUbuntu_12.04
    xUbuntu_14.04
    xUbuntu_14.10
    xUbuntu_15.04
    xUbuntu_15.10 (from 3.7.3)


source s_nginx {
        #pipe ("/proc/kmsg" program_override("kernel: "));
        file("/opt/logs/nginx/access/log.pipe");
        #unix-stream ("/dev/log");
        #internal();
        # udp(ip(0.0.0.0) port(514));
        # tcp(ip(0.0.0.0) port(514));
};

template t_templ { template("$YEAR-$MONTH-$DAY $HOUR:$MIN:$SEC\t$FULLHOST\t$MSGHDR$MSG\n");};

destination d_log_access {file("/opt/logs/nginx/log" perm(0755) template(t_templ)); };

log { source(s_nginx);  destination(d_log_access);};

## 配置
建议看看文档，syslog-ng很灵活。源和目的支持socket流、tcp、udp、pipo、file等，还可以配置策略和流程，比如含有error的日志发给程序处理，把日志直接发到kafka等。

简单配置例子：

	source s_nginx {
        #pipo ("/proc/kmsg" program_override("kernel: "));
        file("/opt/logs/nginx/access/log.pipe");
        #unix-stream ("/dev/log");
        #internal();
        # udp(ip(0.0.0.0) port(514));
	};
	#template t_templ { template("$MSG\n");};
	template t_templ { template("$YEAR-$MONTH-$DAY $HOUR:$MIN:$SEC\t$FULLHOST\t$MSGHDR$MSG\n");};

	#destination d_mlal { usertty("*"); };
	destination d_log_access {file("/opt/logs/nginx/log" perm(0755) template(t_templ)); };

	log { source(s_nginx);  destination(d_log_access);};

## 入门资料

1. http://www.cnblogs.com/lexus/archive/2012/04/24/2467841.html

2. http://www.linuxfly.org/read.php?171
