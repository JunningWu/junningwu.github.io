---
layout: post
title: Building a working GCC+LLVM+Newlib Toolchain on Windows for RISC-V
date: 2020-04-24
updated: 2020-06-02
categories: Tech
tags: [Essay,Tech]
description: Building a working GCC+LLVM+Newlib Toolchain on Windows for RISC-V，主要介绍一下如何在Windows上面编译一个可用的LLVM for RISC-V工具链，包括riscv-gcc和riscv-newlib，从源码进行编译。
---

之前的工作都是在ubuntu上面进行，可能这也是众多个人开发者比较喜欢的平台，而且照着riscv-gnu-toolchain的Readme，基本上也不会遇到什么问题。

之前的一个帖子《Building a working GCC+LLVM+Newlib Toolchain on CentOS8 for RISC-V》，是针对如何使用CentOS来编译生成一个RISC-V可用的工具链。

首先，安装MSYS2，可以在http://msys2.github.io/，我下载的是64-bit版本的msys2-x86_64-20161025.exe。

安装完成MSYS2以后，需要安装一些编译riscv-gnu-toolchain的软件和库，包括mingw-w64-x86_64-toolchain，使用如下命令：

```
pacman -S mingw-w64-x86_64-toolchain --disable-download-timeout
```

为了加快软件和库的下载速度，建议通过修改msys64\etc\pacman.d\mirrorlist.mingw64和msys64\etc\pacman.d\mirrorlist.msys改成国内清华大学的源：

```
Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/x86_64
```

安装完成toolchain以后，可以执行gcc.exe -v查看当前版本，我这边的GCC版本是8.3.0版本，满足要求。

```
Administrator@WIN-M2UUSEE916N MINGW64 ~
# gcc.exe -v
Using built-in specs.
COLLECT_GCC=D:\msys64\mingw64\bin\gcc.exe
COLLECT_LTO_WRAPPER=D:/msys64/mingw64/bin/../lib/gcc/x86_64-w64-mingw32/8.3.0/lto-wrapper.exe
Target: x86_64-w64-mingw32
Configured with: ../gcc-8.3.0/configure --prefix=/mingw64 --with-local-prefix=/mingw64/local --build=x86_64-w64-mingw32 --host=x86_64-w64-mingw32 --target=x86_64-w64-mingw32 --with-native-system-header-dir=/mingw64/x86_64-w64-mingw32/include --libexecdir=/mingw64/lib --enable-bootstrap --with-arch=x86-64 --with-tune=generic --enable-languages=ada,c,lto,c++,objc,obj-c++,fortran --enable-shared --enable-static --enable-libatomic --enable-threads=posix --enable-graphite --enable-fully-dynamic-string --enable-libstdcxx-filesystem-ts=yes --enable-libstdcxx-time=yes --disable-libstdcxx-pch --disable-libstdcxx-debug --disable-isl-version-check --enable-lto --enable-libgomp --disable-multilib --enable-checking=release --disable-rpath --disable-win32-registry --disable-nls --disable-werror --disable-symvers --with-libiconv --with-system-zlib --with-gmp=/mingw64 --with-mpfr=/mingw64 --with-mpc=/mingw64 --with-isl=/mingw64 --with-pkgversion='Rev2, Built by MSYS2 project' --with-bugurl=https://sourceforge.net/projects/msys2 --with-gnu-as --with-gnu-ld
Thread model: posix
gcc version 8.3.0 (Rev2, Built by MSYS2 project)

```

安装完成toolchain以后，还需要安装一些必须的库，包括autoconf，bison，diffutils，make，texinfo，man，cmake，mpc，mpfr-devel，gmp，zlib，expat，isl，使用如下命令：

```
pacman -S mpc mpfr-devel gmp-devel patchutils zlib-devel expat --disable-download-timeout
pacman -S isl --disable-download-timeout
pacman -S git --disable-download-timeout
pacman -S man cmake ninja autoconf bison diffutils make tar texinfo flex --disable-download-timeout
```

好了，这就将编译riscv-gnu-toolchain的所有依赖库安装完成；需要用到的源代码包括下面的三个，llvm-project，riscv-gnu-toolchain，riscv-newlib：

-llvm-project-llvmorg-10.0.0.tar.gz
-riscv-gnu-toolchain-20191225-2a2def26e9ab4ce24ef4266ad5d2f3fd8bd20169.tar.gz
-riscv-newlib-r20200403.tar.gz

## 编译RISC-V GCC

按照RISCV的Readme，我选择的是rv32imc，ilp32，

```
cd riscv-gnu-toolchain
./configure --prefix=/home/Administrator/work/HX2000-Toolchain/riscv-tc-gcc --with-arch=rv32imc --with-abi=ilp32
make -j2 2>&1 | tee make_gcc.log
```

之后，就可以正确编译riscv-tc-gcc的工具链了，编译好以后，设置一下bashrc中的PATH路径。

```
Administrator@WIN-M2UUSEE916N MINGW64 ~/work/HX2000-Toolchain/riscv-tc-gcc/bin
# ./riscv32-unknown-elf-gcc.exe --version
riscv32-unknown-elf-gcc.exe (GCC) 9.2.0
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.


Administrator@WIN-M2UUSEE916N MINGW64 ~/work/HX2000-Toolchain/riscv-tc-gcc/bin
# file riscv32-unknown-elf-gcc.exe
riscv32-unknown-elf-gcc.exe: PE32+ executable (console) x86-64, for MS Windows
```

###编译RISC-V Newlib

可以直接在MSYS2里面编译RISCV-newlib。

由于LLVM的仓库中，没有集成riscv-newlib的代码，因此需要手动编译出来，否则会找不到libm,libc,libgloss等库，我的路径设置的跟llvm的安装路径（INSTALL_DIR环境变量）一样。

```
% git clone https://github.com/riscv/riscv-newlib.git
% cd riscv-newlib 
% mkdir build
% cd build
% ../configure --target=riscv32-unknown-elf --prefix=$INSTALL_DIR
% make all
% make install
```

编译完成以后，记得分别设置一下lib和include的路径，在编译选项中添加进去。

## 编译RISC-V LLVM

编译Windows的LLVM，在MSYS2下面会报错，尝试了多种方式，依然不行。

我试着在cygwin下面编译，也是会出错，因此采用Visual Studio 2017来编译LLVM。下载完成VS2017的安装包，安装以后，还需要安装cmake和python，直接在相关网站下载最新的安装包即可。我这边使用的cmake是cmake-3.17.1-win64-x64.msi，python是python-3.8.2-amd64.exe。

如果安装完成vs2017以后是中文版本，可以修改成英文版本，这样能够更好的对应上llvm官方的说明。打开cmake gui界面，设置llvm源代码的路径和build的路径。llvm源代码路径为llvm-project/llvm，我之前build的路径是在根目录下，因此是llvm-project/build。

**更新于20210420**安装完成vs2017或者vs2019之后，还不能直接与cmake-gui结合来编译clang，还需要设置环境变量，包括path，lib，include，如下：

PATH:
```
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.26.28801\bin\Hostx86\x64
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.26.28801\bin\Hostx86\x86
```
INCLUDE:
```
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.26.28801\include
C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\ucrt
```
LIB:
```
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.26.28801\lib\x64
C:\Program Files (x86)\Windows Kits\10\Lib\10.0.18362.0\ucrt\x86
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.26.28801\lib\x86
C:\Program Files (x86)\Windows Kits\10\Lib\10.0.18362.0\ucrt\x64
```


之后，就是设置一下一些编译的标志，如
```
CMAKE_INSTALL_PREFIX (安装路径，我这里设置的是work/HX2000-Toolchain/riscv-tc-llvm)
CMAKE_BUILD_TYPE （其实这里设置成Release后，后面还需要在VS2017中进行设置的）
LLVM_ENABLE_PROJECTS （我这里只需要用到clang，llc，lld等，可以根据需要进行编译）
DEFAULT_SYSROOT （HX2000-Toolchain/riscv-tc-gcc/riscv32-unknown-elf） 
GCC_INSTALL_PREFIX （HX2000-Toolchain/riscv-tc-gcc） 
LLVM_DEFAULT_TARGET_TRIPLE （riscv32-unknown-elf）
LLVM_TARGETS_TO_BUILD （选择RISCV即可）
-Thost=x64（选择64位版本）
```

之后，点击config和generate，就会在build目录下生成一个llvm.sln的工程文件，双击就可以打开VS2017的界面，在VS2017中选择Release之后，选择INSTALL，然后等着就行，结束以后，会在指定的安装目录下看到编译好的llvm。应该没有什么问题，至少我这边没有遇到。

至此（2020-04-24， Beijing Xigema Building）。

(以下更新于20200602，Beijing Xigema Building)

上述编译过程，会依赖于mscvp140.dll和vcruntime140.dll两个动态链接库，而在一些版本的操作系统中可能发生由于这些库文件丢失而无法运行的错误。

因此，通过cmake-gui修改LLVM的配置信息，具体参数为LLVM_USE_CRT_DEBUG, LLVM_USE_CRT_MINISIZEREL, LLVM_USE_CRT_RELEASE, LLVM_USE_CRT_RELWITHDEBINFO, 都选择为MT，

```
Using Debug VC++ CRT: MT
Using Release VC++ CRT: MT
Using MinSizeRel VC++ CRT: MT
Using RelWithDebInfo VC++ CRT: MT
Using Release VC++ CRT: MT
```
这样编译出来的可执行文件会大，但是不会依赖于上述的库文件。
