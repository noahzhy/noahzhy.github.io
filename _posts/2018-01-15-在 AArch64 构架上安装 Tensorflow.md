---
layout:     post
title:      在 AArch64 构架上安装 Tensorflow
subtitle:   简单几条命令，即可实现在 AArch64 构架上安装 Tensorflow
date:       2018-01-15
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Tensorflow
- Orange Pi
- 中文
- 原创
---
# 在 AArch64 构架上安装 Tensorflow
## 运行环境
* Orange Pi PC2
* ARMBIAN 5.37.180112
* Ubuntu 16.04.3 LTS 4.14.13-sunxi64
* Python 3.5.2
* gcc 5.4.0 20160609

## 开始之前
### 更新源及系统
```sh
apt update && apt upgrade -y
```
### 安装 pip3
```sh
apt install python3-pip
```
### 升级 pip3
```sh
pip3 install --upgrade pip
```
### 安装 python3-dev
```sh
apt install python3-dev
```
## 最后一步
### 下载 wheel
```sh
curl -L https://github.com/noahzhy/tf-aarch64/raw/master/tensorflow-1.4.0rc0-cp35-cp35m-linux_aarch64.whl
```
### 安装
```sh
python3 -m pip install tensorflow-1.4.0rc0-cp35-cp35m-linux_aarch64.whl
```

#### 另外：
在整理资料的时候，发现了一个名为[蓝色圣诞](http://bluexmas.tistory.com/category/OS/Orange%20PI)的 Tistory 博客，有兴趣的可以去逛逛。不翻墙的话，有些图片可能会加载不出来。至于韩语什么的，看不懂的就用 Google 翻译吧。反正我感觉就算我一个能看懂韩语的，主要还是看 Code。
