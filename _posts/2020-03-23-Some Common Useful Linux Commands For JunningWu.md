---
layout: post
title: Some Common Useful Linux Commands For JunningWu
date: 2020-03-23
categories: Tech
tags: [Essay,Tech]
description: 这个帖子整理一些经常用到的Linux命令以及一些与Linux相关的操作，便于查询。
---

## 终端内容重定向到文件：2>&1

经常需要调试程序，特别是对于大型程序，如LLVM编译过程，刷屏是常事；而对于编译程序如果打开verbose的话，也需要将log内容重定向到文件中，便于调试。

## 查找命令：find

```
find ./llvm/lib/Target/RISCV/ | xargs grep isUImm5 2>&1 | tee find.log
```

## 打包patch和diff命令

单个文件
```

diff –uN  from-file  to-file  >to-file.patch

patch –p0 < to-file.patch

patch –RE –p0 < to-file.patch
```

多个文件
```

diff –uNr  from-docu  to-docu  >to-docu.patch

patch –p1 < to-docu.patch

patch –R –p1 <to-docu.patch
```

## 查看linux工作目录的大小

```

sudo du -sh /home/*

```


## Ubuntu 18.04和Ubuntu 20.04安装思科anyconnect

```
sudo apt-get install network-manager-openconnect-gnome
```


## 如何在ubuntu 18.04上面安装低版本的gcc和g++

```
g++ --version
sudo apt update
sudo apt upgrade 
sudo apt install gcc-4.8
sudo apt install g++-4.8
```


## 修改或者添加软连接

```
sudo rm /usr/bin/gcc
sudo ln -s /usr/bin/gcc-7 /usr/bin/gcc
sudo rm /usr/bin/g++
sudo ln -s /usr/bin/g++-7 /usr/bin/g++
```