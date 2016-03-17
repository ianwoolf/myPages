+++
date = "2016-02-20T16:30:09+08:00"
draft = true
title = "raspberryPi -- 初始化sd卡"

+++
树莓派的sd卡，需要初始化一下。才能使用全部大小，否则只能使用部分容量。


忘记参考哪个了，有一个有点小问题。自己试下吧

http://blog.csdn.net/lichao_ustc/article/details/46740443
https://www.phpbulo.com/archives/467.html

PS:磁盘操作命令操作不当可能会引起数据丢失，无论有没有把握都必须备份重要的数据。

    sudo fdisk /dev/mmcblk0
    按P
    将看到的分区复制下来/dev/mmcblk0p2的start值，122880。下面会用到。
    执行命令：d  （删除分区2,选择2）
    执行命令：p （按这时候应该是少了一个分区了）
    执行命令：n  (加分区)
    执行命令：p （主要分区）
    选择2
    在开始位置输入start的值，如下图122880，看下图
    后面的值默认即可
    执行命令：p
    执行命令：w
    成功后如下图
    然后我们重启树莓派

重启后登录SSH执行如下命令。`sudo resize2fs /dev/mmcblk0p2`用于修复分区。

执行成功后，再次df看看。

