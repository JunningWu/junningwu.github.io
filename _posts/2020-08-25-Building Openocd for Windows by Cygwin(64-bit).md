---
layout: post
title: Building Openocd for Windows by Cygwin(64-bit)
date: 2020-08-25
updated: 2020-10-19
categories: Tech
tags: [Essay,Tech]
description: 尽管OpenOCD官网也提供了不同操作系统的bin文件，但是，可能很多时候需要根据自家的芯片进行定制化移植，需要自己从源码编译OpenOCD。这个帖子记录一下用Cygwin第一次从源码编译的过程。编译的bin文件，已通过FPGA验证。
---

1.首先需要安装Cygwin，当然也可以选择其他工具，MinGW或者Msys2。我这边使用的时Windows 10 OS。https://cygwin.com/setup-x86_64.exe

在安装的时候，需要联网，选择 "Install from Internet"，同时需要选择下列安装包，可以在搜索框中搜索相关的安装包，安装最新版本的就可以。

- autobuild
- autoconf
- autoconf-archive
- automake
- git
- gcc-core
- libtool
- libusb1.0
- libusb1.0-devel
- make
- pkg-config
- usbutils

选择好上面的包之后，安装Cygwin就可以了。

2.克隆openocd的源码

```
$ mkdir -p ~/build; cd ~/build
$ git clone https://git.code.sf.net/p/openocd/code  openocd
$ cd ~/build/openocd/
$ ./bootstrap
```

如果选择riscv官方支持的版本，可以使用下面的仓库链接

```
https://github.com/riscv/riscv-openocd.git
```

3.配置

```
$ mkdir build
$ cd build
$ ../configure --enable-ftdi
...
OpenOCD configuration summary
--------------------------------------------------
MPSSE mode of FTDI based devices        yes
ST-Link JTAG Programmer                 yes (auto)
TI ICDI JTAG Programmer                 yes (auto)
Keil ULINK JTAG Programmer              yes (auto)
Altera USB-Blaster II Compatible        yes (auto)
Bitbang mode of FT232R based devices    yes (auto)
Versaloon-Link JTAG Programmer          yes (auto)
TI XDS110 Debug Probe                   yes (auto)
OSBDM (JTAG only) Programmer            yes (auto)
eStick/opendous JTAG Programmer         yes (auto)
Andes JTAG Programmer                   yes (auto)
USBProg JTAG Programmer                 no
Raisonance RLink JTAG Programmer        no
Olimex ARM-JTAG-EW Programmer           no
CMSIS-DAP Compliant Debugger            no
Cypress KitProg Programmer              no
Altera USB-Blaster Compatible           no
ASIX Presto Adapter                     no
OpenJTAG Adapter                        no
SEGGER J-Link Programmer                yes (auto)

```

4.编译+安装

```
$ make
$ make install
$ which openocd
/usr/local/bin/openocd
```

5.复制和拷贝可执行文件，需要注意的是，尽管openocd默认的是静态编译，但是，依然需要拷贝两个库文件。cygusb-1.0.dll和cygwin1.dll放到bin的目录下。


（2020-08-25，希格玛公寓，北京）