+++
date = "2016-02-15T16:30:09+08:00"
draft = true
title = "【原创】raspberryPi -- 无线网卡的设置"

+++

将无线网卡插入，lsusb看到usb有新增。ifconf新增了wlan0网卡。之后配置无线网络

`sudo vi /etc/network/interfaces`,修改后文件内容如下：

    auto lo
    iface lo inet loopback
    iface eth0 inet dhcp
    auto wlan0
    allow-hotplug wlan0
    iface wlan0 inet dhcp
    wpa-ssid “你的wifi名称”
    wpa-psk “你的wifi密码”

具体各行配置的意思如下:
>auto lo //表示使用localhost

>iface eth0 inet dhcp //表示如果有网卡ech0, 则用dhcp获得IP地址 (这个网卡是本机的网卡，而不是WIFI网卡)

>auto wlan0 //表示如果有wlan设备，使用wlan0设备名

>allow-hotplug wlan0 //表示wlan设备可以热插拨

>iface wlan0 inet dhcp //表示如果有WLAN网卡wlan0 (就是WIFI网卡), 则用dhcp获得IP地址

>wpa-ssid “你的wifi名称”//表示连接SSID名

>wpa-psk “你的wifi密码”//表示连接WIFI网络时，使用wpa-psk认证方式，认证密码

上述定义后，如果有网线连接，则采取DHCP自动连接获得地址，重启网络设备`sudo /etc/init.d/networking restart`
成功后，用 ifconfig 命令可以看到 wlan0 设备，且有了IP地址(已连接)
或者这样重启wlan0，推荐这样

    sudo ifdown wlan0
    sudo ifup wlan0

之后你可以通过vpn连接到外网，这样就可以访问树莓派了。具体看我博客吧
