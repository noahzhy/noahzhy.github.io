---
layout:     post
title:      Orange Pi PC 2开发板试用体验
subtitle:   简单介绍一下Orange Pi PC 2开发板试用体验心得。
date:       2017-01-06
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Orange Pi
- 中文
---
# Orange Pi PC 2开发板试用体验
## 开始之前
香橙派PC2是不能通过MicroUSB来供电的，所以如果使用USB供电，一定保证电流的足够输入，特别是你的GPIO设备或USB设备比较多的时候。Linux系统的安装方法相似，Android的话略有不同，本文仅以Ubuntu为例详细说明一下如何安装。

### 所需材料
* 香橙派 PC 2 一枚  
* 电脑一台（ Windows、MAC、Linux 均可，本文以 Windows 为例）  
* TF 卡一张（建议 Class 10 及以上，8G 以上）  
* TF 读卡器一个（你电脑自带的话，这个也不需要）  

### 步聚
1. 去官方下载你想安装的操作系统，注意选择与你硬件对应的系统。香橙派 PC 2 官方提供了 Android 系统和 Linux 系统（Debian、Ubuntu、Arch）
2. 下载 TF 卡格式化工具格式化TF卡，个人比较偏好直接在磁盘管理里格式化
3. 下载 Win32 Diskimager，安装并打开，选择你下载的系统映像的 img 文件，写入
4. 写入完成后，将 TF 卡插入香橙派的卡槽，连接电源，开机，会看到登录画面，用户名 root，密码 orangepi
5. 打开终端，看了一下磁盘空间,显示的如果并不是你 TF 卡的容量，这里就得需要扩容至你本身容量，如果用的是 Armbian 的系统的话会自动扩容  

查看磁盘空间命令：
```sh
sudo df -h
```

## 磁盘扩容
### 方案一：图形界面扩容
扩容方法有图形操作和命令行操作两种，图形操作的话，只要安装 Gparted 后，打开操作就行了。  
![](https://github.com/noahzhy/noahzhy.github.io/blob/master/img/df-h.jpg?raw=true)

### 方案二：命令行扩容
这里重点介绍一下命令行的操作。  
命令行操作必需使用 ROOT 权限，按下面说明执行命令。

```sh
$ cat /sys/block/mmcblk0/mmcblk0p2/start   # 查看第二分区的起始地址，记住这个数字
122880
$ fdisk /dev/mmcblk0
Command (m for help): d   #d，删除分区
Partition number (1-4): 2   # 2，删除第二分区

Command (m for help): n  #创建一个新分区

Select (default p): p  #创建主分区

Partition number (1-4, default 2): 2  #分区2
First sector (2048-7744511, default 2048): 122880  #输入第一次得到的第二分区起始地址

Last sector, +sectors or +size{K,M,G} (122880-xxx, default xxx):  #最后一个sector，默认即可，直接按回车
Command (m for help): w   #将上面的操作写入分区表
```
然后可能会告诉你分区繁忙，没关系，直接敲 reboot 重启系统。  
重启完成之后，此时查询也还是没有变化的，还需要如下命令：

```sh
$ resize2fs /dev/mmcblk0p2
```

## 收尾工作
此时再查看你的分区大小，和你的TF卡一样了。系统到这里就算安装完成了，还有一些事情是要做的。  

1. 设置系统时区为上海时区  

```sh
$ timedatectl set-timezone 'Asia/Shanghai'
```

2. 更新源及系统  

```sh
apt update && apt upgrade -y
```

#### 参考：<http://www.ickey.cc/community/thread-74411-1-1.html>
