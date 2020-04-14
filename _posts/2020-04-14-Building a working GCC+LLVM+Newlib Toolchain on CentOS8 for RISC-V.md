---
layout: post
title: Building a working GCC+LLVM+Newlib Toolchain on CentOS8 for RISC-V
date: 2020-04-14
categories: Tech
tags: [Essay,Tech]
description: Building a working GCC+LLVM+Newlib Toolchain on CentOS8 for RISC-V，主要介绍一下如何在CenOS 8上面编译一个可用的LLVM for RISC-V工具链，包括riscv-gcc和riscv-newlib，从源码进行编译。
---

之前的工作都是在ubuntu上面进行，可能这也是众多个人开发者比较喜欢的平台，而且照着riscv-gnu-toolchain的Readme，基本上也不会遇到什么问题。

CentOS8自带的GCC版本是8.3.1版本，满足要求。

-llvm-project-llvmorg-10.0.0.tar.gz
-riscv-gnu-toolchain-20191225-2a2def26e9ab4ce24ef4266ad5d2f3fd8bd20169.tar.gz
-riscv-newlib-r20200403.tar.gz

## 编译RISC-V GCC

按照riscv-gnu-toolchain的说明，直接运行下列命令进行安装依赖库，是会存在问题的，有几个库没有安装，而且有几个库没有安装devel版本，在后面编译的时候，会报错。

```
sudo yum install autoconf automake python3 libmpc-devel mpfr-devel gmp-devel gawk  bison flex texinfo patchutils gcc gcc-c++ zlib-devel expat-devel
```

### libmpc-devel

这个库会比较麻烦，需要手动安装，下载好gmp-c++，gmp-devel，mpfr-devel，以及libmpc-devel的RPM包，进行安装。

### texinfo

这个库可以采用dnf --enablerepo=PowerTools install texinfo进行安装。

### zlib-devel和expat-devel

这两个库可以直接用yum安装。

之后，就可以正确编译riscv-tc-gcc的工具链了，编译好以后，设置一下PATH路径。

## 编译RISC-V LLVM

首先安装cmake和git，这个可以直接用yum安装。

之后，安装一下ninja，这个可以用命令dnf --enablerepo=PowerTools install ninja。

之后编译LLVM应该没有什么问题，至少我这边没有遇到。

###编译RISC-V Newlib

由于LLVM的仓库中，没有集成riscv-newlib的代码，因此需要手动编译出来，否则会找不到libm,libc,libgloss等库，我的路径设置的跟llvm的安装路径一样。

```
% git clone https://github.com/riscv/riscv-newlib.git
% cd riscv-newlib 
% mkdir build
% cd build
% ../configure --target=riscv64-unknown-elf --prefix=$INSTALL_DIR
% make all
% make install
```

编译完成以后，记得分别设置一下lib和include的路径，在编译选项中添加进去。

至此