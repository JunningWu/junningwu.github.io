---
layout: post
title: 安装一个可用的Linux版本Modelsim
date: 2019-12-11
categories: 技术文章
tags: [随笔,技术文章]
description: Linux版本Modelsim，Intel FPGA Edition 10.5b，Ubuntu 16.04虚拟机，VMware，不需要license。
---

## 工具下载
名称：ModelSimSetup-16.1.0.196-linux.run
下载链接：[ModelSim](https://drive.google.com/file/d/0BxghKvvmdklCSm0yTFJJYjNYQXM/view?usp=sharing)

## 安装依赖库及Modelsim
因为我是在64位系统上面安装，所以需要安装ia32的库。安装过程中，选择starter版本，不需要license。
```
udo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install build-essential
sudo apt-get install gcc-multilib g++-multilib lib32z1 lib32stdc++6 lib32gcc1 expat:i386 fontconfig:i386 libfreetype6:i386 libexpat1:i386 libc6:i386 libgtk-3-0:i386 libcanberra0:i386 libpng12-0:i386 libice6:i386 libsm6:i386 libncurses5:i386 zlib1g:i386 libx11-6:i386 libxau6:i386 libxdmcp6:i386 libxext6:i386 libxft2:i386 libxrender1:i386 libxt6:i386 libxtst6:i386
chmod +x ModelSimSetup-16.1.0.196.run
./ModelSimSetup-13.1.0.162.run
```
## 修改环境变量
```
export PATH=$PATH:/home/llvm/intelFPGA/16.1/modelsim_ase/linuxaloem
```

## 启动效果
（。。。。）