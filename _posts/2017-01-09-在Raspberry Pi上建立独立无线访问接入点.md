---
layout:     post
title:      
subtitle:   树莓派可以作为一个无线接入点，运行一个独立的网络。
date:       2018-01-09
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Raspberry Pi
- 中文
- 翻译
---
# 在 Raspberry Pi 上建立独立无线访问接入点
## 准备工作
树莓派可以作为一个无线接入点，运行一个独立的网络。 可以使用 Raspberry Pi 3 或 Raspberry Pi Zero W 的内置无线功能，或者使用支持接入点的合适的 USB 无线网卡来完成此操作。  
  
需要注意的是，本文已在 Raspberry Pi 3 上进行了测试，有些 USB 无线网卡可能需要对其设置稍作更改。 如果你在使用 USB 无线网卡时遇到问题，请查看论坛。  
  
要将基于树莓派的接入点添加到现有的网络，[参考本节](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md#internet-sharing)。 为了可以作为接入点运行，Raspberry Pi 将需要安装接入点软件以及 DHCP 服务器软件，为连接设备提供网络地址。 确保你的 Raspberry Pi 使用最新版本的 Raspbian（日期为2017或更高版本）。   
使用以下命令来更新你的 Raspbian 安装器：
```sh
sudo apt-get update
sudo apt-get upgrade
```
使用此命令一次安装所有必需的软件：
```sh
sudo apt-get install dnsmasq hostapd
```
由于配置文件尚未准备就绪，请关闭新软件，如下所示：
```sh
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```
## 配置一个静态 IP 地址 
我们正在配置一个独立的网络作为服务器，所以 Raspberry Pi 需要有一个分配给无线端口的静态 IP 地址。 本文假定我们为我们的无线网络使用标准的```192.168.x.x``` IP 地址，因此我们将分配给服务器 IP 地址```192.168.0.1```。 还要设置正在使用的无线设备为```wlan0```。  
  
配置静态 IP 地址，使用以下命令编辑 dhcpcd 配置文件：
```sh
sudo nano /etc/dhcpcd.conf
```

转到该文件的末尾并对其进行编辑，使其如下所示：
```sh
interface wlan0
    static ip_address=192.168.4.1/24
```
现在重新启动 dhcpcd 进程并设置新的```wlan0```配置：
```sh
sudo service dhcpcd restart
```

## 配置 DHCP 服务器（dnsmasq）
DHCP 服务由 dnsmasq 提供。 默认情况下，配置文件包含很多不需要的信息，并且从零开始更容易些。 重命名这个配置文件，并编辑一个新的：
```sh
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig  
sudo nano /etc/dnsmasq.conf
```
将以下信息输入或复制到 dnsmasq 配置文件中并保存：
```sh
interface=wlan0      # 使用所需的无线接口，通常是 wlan0
  dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```
因此对于```wlan0```，我们将提供```192.168.4.2```和```192.168.4.20``之间的 IP 地址，租用时间为24小时。 如果你正在为其他网络设备（例如 eth0）提供 DHCP 服务，则可以在配置文件头部适当添加更多的接口，以及你打算提供给该接口的地址范围。  
  
dnsmasq 还有很多选项。 有关更多详细信息，请参考 [dnsmasq 文档](http://www.thekelleys.org.uk/dnsmasq/doc.html)。


## 配置接入点 host 软件（hostapd）
你需要编辑位于```/etc/hostapd/hostapd.conf```的 hostapd 配置文件，为你的无线网络添加各种参数。 初次安装后，这可能是一个新文件或者空文件。
```sh
sudo nano /etc/hostapd/hostapd.conf
```
将以下信息添加到配置文件中。 此配置中我们假设使用的是第7频道，网络名称为```NameOfNetwork```，密码为```AardvarkBadgerHedgehog```。 注意名称和密码不应有引号。
```sh
interface=wlan0
driver=nl80211
ssid=NameOfNetwork
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=AardvarkBadgerHedgehog
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```
现在我们需要告诉系统在哪里找到这个配置文件。
```sh
sudo nano /etc/default/hostapd
```
查找```#DAEMON_CONF```这行，并用下面的代替它：
```sh
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

## 启动它
现在启动其余的服务：
```sh
sudo service hostapd start
sudo service dnsmasq start
```

## 添加路由和地址伪装
编辑```/etc/sysctl.conf```并取消这一行的注释：
```sh
net.ipv4.ip_forward=1
```
为```eth0```出站流量添加伪装：
```sh
sudo iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
```
保存 iptables 规则。
```sh
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
编辑```/etc/rc.local```并在```exit 0```上方添加这个以在启动时安装这些规则。
```sh
iptables-restore < /etc/iptables.ipv4.nat
```
重启  
  

使用无线设备搜索网络。 现在应该存在了你在 hostapd 配置中指定的网络 SSID，并且应该可以使用指定的密码进行访问。 如果在 Raspberry Pi 接入点上启用了 SSH，应该可以通过如下方式从另一个 Linux 机器（或具有 SSH 连接功能的系统）连接到它，假设为```pi``帐户：
```sh
ssh pi@192.168.4.1
```
在这点上，树莓派是作为一个接入点，其他设备可以与它连接。相关设备可以通过其 IP 地址访问 Raspberry Pi 接入点，以执行诸如```rsync```，```scp```或```ssh```等操作。

## 使用树莓派作为接入点共享互联网连接
Raspberry Pi 作为接入点的一个常见用途是为有线以太网提供无线连接，以便登录到接入点的任何人都可以访问互联网，当然，Pi 上的有线以太网可以通过某种路由器连接到互联网。 为此，需要在无线设备和接入点 Raspberry Pi 上的以太网设备之间建立一个“桥”。 这个桥将通过两个接口之间的所有流量。 安装以下软件包以启用接入点设置和桥接。
```sh
sudo apt-get install hostapd bridge-utils
```
由于配置文件尚未准备就绪，先关闭新装的软件，如下所示：
```sh
sudo systemctl stop hostapd
```
桥接器在两个被桥接的端口上创建了一个更高级别的的结构。 它是网络设备的桥梁，所以我们需要停止 Raspberry Pi 上 DHCP 客户端分配 IP 地址的```eth0```和```wlan0```端口。
```sh
sudo nano /etc/dhcpcd.conf
```
将```denyinterfaces wlan0```和```denyinterfaces eth0```添加到文件的末尾（但在其他添加的```interface```接口之上）并保存该文件。   
在这个例子中添加一个新的名为```br0```的桥。
```sh
sudo brctl addbr br0
```
连接网络端口。在这个例子中，将```eth0```连接到桥接器```br0```。
```sh
sudo brctl addif br0 eth0
```
现在需要编辑接口文件来调试各种设备以使用桥接。 
```sh
sudo nano /etc/network/interfaces
```
进行以下编辑，在文件末尾添加桥接信息。
```sh
# Bridge setup
auto br0
iface br0 inet manual
bridge_ports eth0 wlan0
```
接入点设置与上一节中显示的几乎相同。按照上面的说明配置```hostapd.conf```文件，但在```interface = wlan0```行下面添加```bridge = br0```，然后删除或注释掉驱动程序这行。
```sh
interface=wlan0
bridge=br0
#driver=nl80211
...
```
现在重启 Raspberry Pi  
  
现在无线局域网和 Raspberry Pi 上的以太网连接之间应该有一个功能上的桥梁，任何与 Raspberry Pi 接入点相连的设备，都会像连接有线以太网到接入点那样工作。  
  

```ifconfig```命令将显示将通过有线以太网的 DHCP 服务器分配的 IP 地址的网桥。 ```wlan0```和```eth0```不再具有 IP 地址，因为它们现在由网桥控制。 如果需要，可以为网桥使用静态 IP 地址，但一般来说，如果 Raspberry Pi 接入点连接到 ADSL 路由器，DHCP 地址将会正常。

#### 原文：<https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md>
