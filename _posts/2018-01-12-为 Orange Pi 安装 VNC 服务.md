---
layout:     post
title:      为 Orange Pi 安装 VNC 服务
subtitle:   为 Orange Pi 安装 xfce4 桌面及 VNC 服务
date:       2018-01-12
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Orange Pi
- 中文
---
# 为 Orange Pi 安装 VNC
## 为 Orange Pi 安装 xfce4 桌面及 VNC 服务
### 安装 xfce4 桌面
```sh
apt-get install xfce4
```

### 安装 vnc 服务
```sh
apt-get install vnc4server
```

### 设置 vnc 密码
安装完成后对vnc4server设置密码：  
```sh
vnc4server
```
  

![](https://github.com/noahzhy/noahzhy.github.io/blob/master/img/vncserver_passwd.jpg?raw=true)  

### 尝试远程
使用 [vnc viewer](https://www.realvnc.com/en/connect/download/viewer/windows/) 远程连接  
  
![](https://github.com/noahzhy/noahzhy.github.io/blob/master/img/vnc_viewer.jpg?raw=true)

### Windows为例打开 vnc viewer 然后输入 ip 与端口
默认端口为5900或5901或1  
  
![](https://github.com/noahzhy/noahzhy.github.io/blob/master/img/vncserver_pc.jpg?raw=true)  

### 远程成功
没有桌面是因为我们还没配置 vnc 启动的桌面  
  
![](https://github.com/noahzhy/noahzhy.github.io/blob/master/img/grey.jpg?raw=true)  
  
## 配置 vnc 启动的桌面
### 进入 .vnc 目录
进入当前终端用户的.vnc目录
```sh
cd ~/.vnc
```

### 备份好原来的 xstartup
```sh
cp xstartup xstartup-bak
```

### 修改 xstartup 文件
```sh
sudo nano xstartup
```

在```x-terminal-emulator -geometry 80x24+10+10 -ls -title "$VNCDESKTOP Desktop" &```
这句之后注释掉最后一句```# x-window-manager &```，然后添加下面这段。  
```sh
x-sesion-manager & xfdesktop & xfce4-panel &
xfce4-menu-plugin &
xfsettingsd &
xfconfd &
xfwm4 &
```
  
![](https://github.com/noahzhy/noahzhy.github.io/blob/master/img/vnc_xstart.jpg?raw=true)

### 重启 vnc 服务
```sh
vncserver -kill :1
vncserver :1
```
  
![](https://github.com/noahzhy/noahzhy.github.io/blob/master/img/desktop.jpg?raw=true)

#### 参考：<https://www.jianshu.com/p/78ed5c90438e>
