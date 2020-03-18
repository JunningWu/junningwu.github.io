---
layout: post
title: 安装一个可用的Linux版本Modelsim
date: 2019-12-11
categories: Tech
tags: [Eaasy,Tech]
description: Linux版本Modelsim，Intel FPGA Edition 10.5b，Ubuntu 16.04虚拟机，VMware，不需要license。
---
## ModelSimSetup-16.1.0
### 工具下载
名称：ModelSimSetup-16.1.0.196-linux.run
下载链接：[ModelSim](https://drive.google.com/file/d/0BxghKvvmdklCSm0yTFJJYjNYQXM/view?usp=sharing)

### 安装依赖库及Modelsim
因为我是在64位系统上面安装，所以需要安装ia32的库。安装过程中，选择starter版本，不需要license。
```
udo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install build-essential
sudo apt-get install gcc-multilib g++-multilib lib32z1 lib32stdc++6 lib32gcc1 expat:i386 fontconfig:i386 libfreetype6:i386 libexpat1:i386 libc6:i386 libgtk-3-0:i386 libcanberra0:i386 libpng12-0:i386 libice6:i386 libsm6:i386 libncurses5:i386 zlib1g:i386 libx11-6:i386 libxau6:i386 libxdmcp6:i386 libxext6:i386 libxft2:i386 libxrender1:i386 libxt6:i386 libxtst6:i386
chmod +x ModelSimSetup-16.1.0.196.run
./ModelSimSetup-13.1.0.162.run
```
### 修改环境变量
```
export PATH=$PATH:/home/junningwu/intelFPGA/16.1/modelsim_ase/linuxaloem
```

### 启动效果
（。。。。）
这个版本的Modelsim可以正常使用，但是博主打算用来仿真Pulpissimo，里面涉及到一些64位的特性，由于刚开始接触这个平台，所以打算再重新安装一个64位的modelsim。

## ModelSimSetup-19.3.0

### 工具下载
名称：ModelSimProSetup-19.3.0.222-linux.run
      ModelSimProSetup-part2-19.3.0.222-linux.run
下载链接：[ModelSim](https://fpgasoftware.intel.com/?edition=pro&platform=linux)

下载完成后，可以直接安装，因为之前已经安装过依赖库，两个安装文件，只需要执行第一个就行：
```
chmod +x ModelSimProSetup-19.3.0.222-linux.run
./ModelSimProSetup-19.3.0.222-linux.run
```
### 修改环境变量
```
export PATH=$PATH:/home/junningwu/intelFPGA/19.3/modelsim_ase/linuxaloem
```
安装完成19.3版本的Modelsim后，经过再重新尝试编译Pulpissimo，博主已经放弃治疗了。（太多坑了），具体来说就是：
- Altera Modelsim Starter版本仅支持32位
- Altera Modelsim Starter版本不支持opt优化
- Altera Modelsim Starter版本对于代码量和模块数目有限制

## 仿真Pulpino

### 克隆仓库并update IPs
```
git clone https://github.com/pulp-platform/pulpino.git
./update-ips.py
```
工具链博主使用的是pulp-riscv-gnu-toolchain，Newlib cross-compiler for Pulp
```
./configure --prefix=/opt/riscv --with-arch=rv32imc --with-cmodel=medlow --enable-multilib
make
```
### 编译pulpino
之后就是在sw目录下，执行
```
mkdir build
cd build
cp ../cmake_configure.riscv.gcc.sh ./
./cmake_configure.riscv.gcc.sh
```
这样会报错，这是因为pulpino本身的问题，并不是工具链的问题，博主之前尝试了riscv的toolchain和pulp的，都存在问题，后来在pulpino的issue里面发现了解决方法[not able to compile a simple test program](https://github.com/pulp-platform/pulpino/issues/281#issuecomment-477085049)。

- fix_m32.sh

```
#!/bin/bash

#Find and replace all occurrances of '-m32' and fix rest of line.

for file in $(find); do
    if [[ -f $file ]]; then
        [[ $(cat $file | grep m32) ]]
        if [[ $? == 0 ]]; then
            echo writing...
            echo $file
           sed 's/\-m32//g' $file > tmp && mv tmp $file
       fi
    fi
done
```

- fix_linker.sh

```
#!/bin/bash

riscv32-unknown-elf-ld --verbose | head -n -1 | tail -n +7 | sed '168 a \ \ _fbss = .;' | sed '169 a \ \ . = .;' > /home/path/to/pulpino/sw/build/CMakeFiles/CMakeTmp/riscv.ld
```

按照issue的经验以及博主的经验，需要多尝试几次，原贴中写的（~5），博主执行到第四次的时候，就过了。
修改march如下：

```
GCC_MARCH="RV32IMXpulpv2"
```

执行效果如下：

```
junningwu@junningwu-vm:~/workspace/pulp/pulpino/sw/build$ ./cmake_configure.riscv.gcc.sh 
-- GCC_MARCH= RV32IMXpulpv2
-- USE_ZERO_RISCY= 0
-- RISCY_RV32F= 0
-- ZERO_RV32M= 0
-- ZERO_RV32E= 0
-- PL_NETLIST= 
-- Configuring done
-- Generating done
-- Build files have been written to: /home/junningwu/workspace/pulp/pulpino/sw/build
```

运行make vcompile，直到运行结果如下，说明编译没有问题。

```
... ... 
... ...
--> PULPino compilation complete! 
--> Compiling work.tb... 
Compiling component:  work.tb 

--> work.tb compilation complete! 

--> PULPino platform compilation complete! 

Built target vcompile
```

### 运行helloworld
当执行make hello_world.vsimc，会报错，unrecognised emulation mode: elflriscv:

```
[ 50%] Linking CXX executable helloworld.elf
/home/junningwu/workspace/pulp/pulp-tc-newlib/lib/gcc/riscv32-unknown-elf/7.1.1/../../../../riscv32-unknown-elf/bin/ld: unrecognised emulation mode: elflriscv
Supported emulations: elf32lriscv elf64lriscv
collect2: error: ld returned 1 exit status
apps/helloworld/CMakeFiles/helloworld.elf.dir/build.make:101: recipe for target 'apps/helloworld/helloworld.elf' failed
make[3]: *** [apps/helloworld/helloworld.elf] Error 1
CMakeFiles/Makefile2:1578: recipe for target 'apps/helloworld/CMakeFiles/helloworld.elf.dir/all' failed
make[2]: *** [apps/helloworld/CMakeFiles/helloworld.elf.dir/all] Error 2
CMakeFiles/Makefile2:956: recipe for target 'apps/helloworld/CMakeFiles/helloworld.vsimc.dir/rule' failed
make[1]: *** [apps/helloworld/CMakeFiles/helloworld.vsimc.dir/rule] Error 2
Makefile:376: recipe for target 'helloworld.vsimc' failed
make: *** [helloworld.vsimc] Error 2
```

在pulpino的issue中，看到有人建议用[ri5cy_gnu_toolchain](https://github.com/pulp-platform/ri5cy_gnu_toolchain) ，这是一个老版本的gnu-toolchain，应该与pulpino的cmake脚本一致。而目前用到的pulp-riscv-gnu-toolchain比较新，有一些性能并不适用于pulpino。

```
in/install/riscv32-unknown-elf/lib; \
  done
make[4]: Leaving directory '/home/junningwu/workspace/pulp/ri5cy_gnu_toolchain/build/build-gcc-newlib/riscv32-unknown-elf/libgloss/riscv'
make[4]: Entering directory '/home/junningwu/workspace/pulp/ri5cy_gnu_toolchain/build/build-gcc-newlib/riscv32-unknown-elf/libgloss/libnosys'
make[4]: Leaving directory '/home/junningwu/workspace/pulp/ri5cy_gnu_toolchain/build/build-gcc-newlib/riscv32-unknown-elf/libgloss/libnosys'
make[3]: Leaving directory '/home/junningwu/workspace/pulp/ri5cy_gnu_toolchain/build/build-gcc-newlib/riscv32-unknown-elf/libgloss'
make[2]: Leaving directory '/home/junningwu/workspace/pulp/ri5cy_gnu_toolchain/build/build-gcc-newlib'
make[1]: Leaving directory '/home/junningwu/workspace/pulp/ri5cy_gnu_toolchain/build/build-gcc-newlib'
```

```
junningwu@junningwu-vm:~/workspace/pulp/ri5cy_gnu_toolchain$ cd install/
bin/                 lib/                 riscv32-unknown-elf/
include/             libexec/             share/
```

然后，再回到sw/build目录下，重新compile和执行，如果modelsim没什么问题，其实应该已经可以运行仿真了。但是，Altera Modelsim不支持64位的版本，所以依然还存在错误。

```
[ 75%] Generating slm_files/l2_ram.slm
[100%] Built target helloworld.slm.cmd
Scanning dependencies of target helloworld.vsimc
[100%] Running helloworld in ModelSim
Failed to open executable /home/junningwu/intelFPGA_pro/19.3/modelsim_ase/linuxaloem/../linux_x86_64pe/vish in execute mode needed for the option -64.
execv: No such file or directory
** Fatal: Unable to exec the GUI /home/junningwu/intelFPGA_pro/19.3/modelsim_ase/linuxaloem/../linux_x86_64pe/vish.
apps/helloworld/CMakeFiles/helloworld.vsimc.dir/build.make:57: recipe for target 'apps/helloworld/CMakeFiles/helloworld.vsimc' failed
make[3]: *** [apps/helloworld/CMakeFiles/helloworld.vsimc] Error 1
CMakeFiles/Makefile2:815: recipe for target 'apps/helloworld/CMakeFiles/helloworld.vsimc.dir/all' failed
make[2]: *** [apps/helloworld/CMakeFiles/helloworld.vsimc.dir/all] Error 2
CMakeFiles/Makefile2:822: recipe for target 'apps/helloworld/CMakeFiles/helloworld.vsimc.dir/rule' failed
make[1]: *** [apps/helloworld/CMakeFiles/helloworld.vsimc.dir/rule] Error 2
Makefile:350: recipe for target 'helloworld.vsimc' failed
make: *** [helloworld.vsimc] Error 2
```

通过修改文件sw/apps/CMakeSim.txt中关于vsim -64的选项，可以正常编译pulpino，但是还是会报错，Error loading design：

```
Scanning dependencies of target helloworld.vsimc
[100%] Running helloworld in ModelSim
Error loading design
apps/helloworld/CMakeFiles/helloworld.vsimc.dir/build.make:57: recipe for target 'apps/helloworld/CMakeFiles/helloworld.vsimc' failed
make[3]: *** [apps/helloworld/CMakeFiles/helloworld.vsimc] Error 12
CMakeFiles/Makefile2:949: recipe for target 'apps/helloworld/CMakeFiles/helloworld.vsimc.dir/all' failed
make[2]: *** [apps/helloworld/CMakeFiles/helloworld.vsimc.dir/all] Error 2
CMakeFiles/Makefile2:956: recipe for target 'apps/helloworld/CMakeFiles/helloworld.vsimc.dir/rule' failed
make[1]: *** [apps/helloworld/CMakeFiles/helloworld.vsimc.dir/rule] Error 2
Makefile:376: recipe for target 'helloworld.vsimc' failed
make: *** [helloworld.vsimc] Error 2
```

然后，就在[issue110](https://github.com/pulp-platform/pulpino/issues/110) 中找到了解决方法，完美地解决了这个问题。至此，pulpino可以正常仿真了。

```
junningwu@junningwu-vm:~/workspace/pulp/pulpino/sw/build$ make helloworld.vsimc
[  0%] Built target bench
[  0%] Built target crt0
[  0%] Built target string
[ 25%] Built target sys
[ 25%] Built target Arduino_core
[ 50%] Built target Arduino_separate
Scanning dependencies of target helloworld.elf
[ 50%] Building C object apps/helloworld/CMakeFiles/helloworld.elf.dir/helloworld.c.o
[ 50%] Linking CXX executable helloworld.elf
[ 50%] Built target helloworld.elf
[ 50%] Generating helloworld.s19
[ 50%] Generating vectors/stim.txt
[ 50%] Built target helloworld.stim.txt
[ 75%] Built target helloworld.links
[ 75%] Generating slm_files/l2_ram.slm
[ 75%] Built target helloworld.slm.cmd
[100%] Running helloworld in ModelSim
!!!!!Hello Haawking!!!!!
[100%] Built target helloworld.vsimc
```
