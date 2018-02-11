---
layout:     post
title:      Raspberry Pi 关闭 AP 热点
subtitle:   关闭 AP 热点，切换回 STA 模式
date:       2018-02-11
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Raspberry Pi
- 原创
---
# Raspberry Pi 关闭 AP 热点
>此方法适用于之前我翻译的文章《[在 Raspberry Pi 上建立独立无线访问接入点](https://noahzhy.github.io/2018/01/09/%E5%9C%A8Raspberry-Pi%E4%B8%8A%E5%BB%BA%E7%AB%8B%E7%8B%AC%E7%AB%8B%E6%97%A0%E7%BA%BF%E8%AE%BF%E9%97%AE%E6%8E%A5%E5%85%A5%E7%82%B9/)》

## 关闭 dnsmasq 和 hostapd
```sh
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

## 修改 DHCP 配置文件
### 关闭之前定义的端口
打开配置文件
```sh
sudo nano /etc/dnsmasq.conf
```
注释掉定义的端口
```sh
# interface=wlan0      # 使用所需的无线接口，通常是 wlan0
#   dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```

## 修改 hostapd 配置文件
### 注释掉配置文件路径
打开配置文件
```sh
sudo nano /etc/default/hostapd
```
修改配置文件，注释掉这行
```sh
# DAEMON_CONF="/etc/hostapd/hostapd.conf"
```
接下来重启
```sh
sudo reboot
```

## 配置无线网络
用编辑器打开```interfaces```文件
```sh
sudo nano /etc/network/interfaces
```

我的```interfaces```文件是这样的：
```sh
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

auto lo
iface lo inet loopback
iface eth0 inet dhcp

allow-hotplug wlan0
iface wlan0 inet static  
    address 192.168.0.1
    netmask 255.255.255.0
    network 192.168.0.0
```

我们把无线网卡部分全部注释掉，然后添加自己的配置信息，最终结果如下：
```sh
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

auto lo
iface lo inet loopback
iface eth0 inet dhcp

auto wlan0
iface wlan0 inet dhcp
wpa-conf /etc/wpa.conf
iface default inet dhcp

# allow-hotplug wlan0
# iface wlan0 inet static  
#     address 192.168.0.1
#     netmask 255.255.255.0
#     network 192.168.0.0
```

使用```nano```编辑器，```ctrl+o```保存，```ctrl+x```退出。

## 配置 Wifi 的 SSID、加密方式和密码
用编辑器```nano```创建```/etc/wpa.conf```文件：
```sh
sudo nano /etc/wpa.conf
```

如果你的wifi没有密码
```sh
network={
    ssid="你的无线网络名称（ssid）"
    key_mgmt=NONE
}
```
如果你的wifi使用WEP加密
```sh
network={
    ssid="你的无线网络名称（ssid）"
    key_mgmt=NONE
    wep_key0="你的wifi密码"
}
```
如果你的wifi使用WPA/WPA2加密
```sh
network={
    ssid="你的无线网络名称（ssid）"
    key_mgmt=WPA-PSK
    psk="你的wifi密码"
}
```

## 最后一步
重启无线网卡：
```sh
sudo ifdown wlan0
sudo ifup wlan0
```
