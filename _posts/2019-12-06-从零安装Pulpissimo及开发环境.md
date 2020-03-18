---
layout: post
title: 从零安装Pulpissimo及开发环境
date: 2019-12-06
categories: Tech
tags: [Eaasy,Tech]
description: Pulp-platform是一个众多RISC-V相关处理器核和SoC平台的集合，由于项目繁多，分支不一，搭建一个能用的平台也没有一个比较容易上手的教程，博主也是反反复复卸载安装了无数个虚拟机，希望这个帖子可以记录一下能够使用的初学者上手的示例。
---

## 使用平台
ubuntu 16.04 VMware虚拟机


## 克隆Pulpissimo的代码
这个没有什么多讲的，直接执行git clone就行。在执行update-ips.py的时候，需要安装yaml，因为用的是python3，所以安装的时候需要指定python版本。
```
git clone https://github.com/pulp-platform/pulpissimo.git
sudo apt-get install python3-yaml
./update-ips
Retrieving ips_list.yml dependency list for all IPs (may take some time)...
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/pulp_soc/master/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/L2_tcdm_hybrid_interco/pulpissimo-v1.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/adv_dbg_if/v0.0.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/apb2per/v0.0.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/apb_adv_timer/v1.0.2/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/apb_fll_if/pulpissimo-v1.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/apb_gpio/v0.2.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/apb_node/v0.1.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/apb_interrupt_cntrl/v0.0.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/axi/v0.7.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/axi_node/v1.1.4/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/axi_slice/v1.1.4/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/axi_slice_dc/v1.1.3/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/axi_mem_if/v0.2.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/timer_unit/v1.0.2/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/common_cells/v1.13.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/fpnew/v0.6.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/jtag_pulp/v0.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/riscv/pulpissimo-v3.4.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/lowRISC/ibex/13313952cd50ff04489f6cf3dba9ba05c2011a8b/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/scm/v1.0.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/generic_FLL/v0.1/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/tech_cells_generic/v0.1.6/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_core/v1.0.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_uart/v1.0.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_i2c/vega_v1.0.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_i2s/v1.0.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_qspi/v1.0.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_sdio/vega_v1.0.5/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_camera/v1.0.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_filter/v1.0.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/udma_external_per/v1.0.0/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/hwpe-mac-engine/v1.2/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/riscv-dbg/v0.2/ips_list.yml
   Fetching ips_list.yml from https://raw.githubusercontent.com/pulp-platform/tbtools/master/ips_list.yml
Generated IP dependency tree.
```
update-ips会下载众多的pulp-platform的IP，包括AXI，APB，RISCV等。如果IP下载没有出错，最后会给出一个总结：
```
SUMMARY
IPs updated successfully!
Generated new scripts for IPs!
```


## 安装Pulp开发工具链
首先安装开发工具链的依赖库，比较奇怪的是，对于configparser分别安装了pip3和pip2两个版本，但是却没有安装python-pip，因此在执行pip2安装的时候，需要先安装python-pip：
```
$ sudo apt-get install git python3-pip gawk texinfo libgmp-dev libmpfr-dev libmpc-dev swig3.0 libjpeg-dev lsb-core doxygen python-sphinx sox graphicsmagick-libmagick-dev-compat libsdl2-dev libswitch-perl libftdi1-dev cmake scons libsndfile1-dev
$ sudo -H pip3 install twisted prettytable pyelftools openpyxl xlsxwriter pyyaml numpy configparser pyvcd
$ sudo apt-get install python-pip
$ sudo pip2 install configparser
```
设置PULP_RISCV_GCC_TOOLCHAIN环境变量，也就是你希望工具链安装在哪里；以及VSIM_PATH环境变量：
```
export PULP_RISCV_GCC_TOOLCHAIN=/home/llvm/workspace/pulp/pulp-tc
export VSIM_PATH=/home/llvm/workspace/pulp/pulpissimo/sim
```
下载pulp-riscv-gnu-toolchain源码，博主自己多次尝试下载，都无功而返，所里网络始终不能完成下载，无奈采取浏览器下载压缩包形式，具体版本信号如下：
```
pulp-riscv-gnu-toolchain-master @ 34c464f
riscv-binutils-gdb @ e3a7346
riscv-dejagnu @ 4ea498a
riscv-gcc @ daa865e
riscv-glibc @ 4e29434
riscv-newlib @ 1e52935

pulp-riscv-binutils-gdb-e3a73468a18de66eadc783b8528eecba05595974.zip
pulp-riscv-gcc-daa865ea46d212dee71872cabd390f700ac2f0c4.zip
pulp-riscv-gnu-toolchain-master.zip
pulp-riscv-newlib-1e52935101d096bb2e9381c7b131d6b976f0acd9.zip
riscv-dejagnu-4ea498a8e1fafeb568530d84db1880066478c86b.zip
riscv-glibc-4e2943456e690d89f48e6e710757dd09404b0c9a.zip
```
安装工具链编译所需的依赖库，configure，make：
```
$ sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev
$ ./configure --prefix=/home/llvm/workspace/pulp/pulp-tc --with-arch=rv32imc --with-cmodel=medlow --enable-multilib
$ make
```
编译完成工具链之后，貌似Pulp的程序编译，还必须借助于SDK，否则需要各位自己把头文件，链接文件啥的都自己添加到gcc的选项中。
```
junningwu@llvm-vm:~/workspace/pulp/test-progs/hello_haawking$ ls /home/llvm/workspace/pulp/pulp-tc/bin/
riscv32-unknown-elf-addr2line  riscv32-unknown-elf-elfedit    riscv32-unknown-elf-gcc-ranlib  riscv32-unknown-elf-ld       riscv32-unknown-elf-readelf
riscv32-unknown-elf-ar         riscv32-unknown-elf-g++        riscv32-unknown-elf-gcov        riscv32-unknown-elf-ld.bfd   riscv32-unknown-elf-run
riscv32-unknown-elf-as         riscv32-unknown-elf-gcc        riscv32-unknown-elf-gcov-dump   riscv32-unknown-elf-nm       riscv32-unknown-elf-size
riscv32-unknown-elf-c++        riscv32-unknown-elf-gcc-7.1.1  riscv32-unknown-elf-gcov-tool   riscv32-unknown-elf-objcopy  riscv32-unknown-elf-strings
riscv32-unknown-elf-c++filt    riscv32-unknown-elf-gcc-ar     riscv32-unknown-elf-gdb         riscv32-unknown-elf-objdump  riscv32-unknown-elf-strip
riscv32-unknown-elf-cpp        riscv32-unknown-elf-gcc-nm     riscv32-unknown-elf-gprof       riscv32-unknown-elf-ranlib
```
此时编译程序，如果不链接，还是可以编译的成功的：
```
junningwu@llvm-vm:~/workspace/pulp/test-progs/hello_haawking$ cat hello_haawking.c
#include <stdio.h>
#include <stdlib.h>

int main(){
  volatile int i,j=0;
  
  for(i = 0; i< 10; i++){
    j += j*i;
  }
   
  return 0;
}

junningwu@llvm-vm:~/workspace/pulp/test-progs/hello_haawking$ $PULP_RISCV_GCC_TOOLCHAIN/bin/riscv32-unknown-elf-gcc --static -v -o hello_haawking -c hello_haawking.c
Using built-in specs.
COLLECT_GCC=/home/llvm/workspace/pulp/pulp-tc/bin/riscv32-unknown-elf-gcc
Target: riscv32-unknown-elf
Configured with: /home/llvm/workspace/pulp/pulp-riscv-gnu-toolchain/riscv-gcc/configure --target=riscv32-unknown-elf --prefix=/home/llvm/workspace/pulp/pulp-tc --disable-shared --disable-threads --enable-languages=c,c++ --with-system-zlib --enable-tls --with-newlib --with-headers=/home/llvm/workspace/pulp/pulp-tc/riscv32-unknown-elf/include --disable-libmudflap --disable-libssp --disable-libquadmath --disable-libgomp --disable-nls --enable-checking=yes --enable-multilib --with-abi=ilp32 --with-arch=rv32imc 'CFLAGS_FOR_TARGET=-Os  -mcmodel=medlow'
Thread model: single
gcc version 7.1.1 20170509 (GCC) 
COLLECT_GCC_OPTIONS='-static' '-v' '-o' 'hello_haawking' '-c' '-march=rv32imc' '-mabi=ilp32'
 /home/llvm/workspace/pulp/pulp-tc/libexec/gcc/riscv32-unknown-elf/7.1.1/cc1 -quiet -v hello_haawking.c -quiet -dumpbase hello_haawking.c -march=rv32imc -mabi=ilp32 -auxbase-strip hello_haawking -version -o /tmp/cc8NqYAI.s
GNU C11 (GCC) version 7.1.1 20170509 (riscv32-unknown-elf)
	compiled by GNU C version 5.4.0 20160609, GMP version 6.1.0, MPFR version 3.1.4, MPC version 1.0.3, isl version none
GGC heuristics: --param ggc-min-expand=30 --param ggc-min-heapsize=4096
#include "..." search starts here:
#include <...> search starts here:
 /home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/include
 /home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/include-fixed
 /home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/../../../../riscv32-unknown-elf/sys-include
 /home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/../../../../riscv32-unknown-elf/include
End of search list.
GNU C11 (GCC) version 7.1.1 20170509 (riscv32-unknown-elf)
	compiled by GNU C version 5.4.0 20160609, GMP version 6.1.0, MPFR version 3.1.4, MPC version 1.0.3, isl version none
GGC heuristics: --param ggc-min-expand=30 --param ggc-min-heapsize=4096
Compiler executable checksum: 1ab3f3c012405b261fe78291777fba0b
COLLECT_GCC_OPTIONS='-static' '-v' '-o' 'hello_haawking' '-c' '-march=rv32imc' '-mabi=ilp32'
 /home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/../../../../riscv32-unknown-elf/bin/as -v --traditional-format -march=rv32imc -mabi=ilp32 -o hello_haawking /tmp/cc8NqYAI.s
GNU assembler version 2.28.0 (riscv32-unknown-elf) using BFD version (GNU Binutils) 2.28.0.20170505
COMPILER_PATH=/home/llvm/workspace/pulp/pulp-tc/libexec/gcc/riscv32-unknown-elf/7.1.1/:/home/llvm/workspace/pulp/pulp-tc/libexec/gcc/riscv32-unknown-elf/7.1.1/:/home/llvm/workspace/pulp/pulp-tc/libexec/gcc/riscv32-unknown-elf/:/home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/:/home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/:/home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/../../../../riscv32-unknown-elf/bin/
LIBRARY_PATH=/home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/:/home/llvm/workspace/pulp/pulp-tc/lib/gcc/riscv32-unknown-elf/7.1.1/../../../../riscv32-unknown-elf/lib/:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-static' '-v' '-o' 'hello_haawking' '-c' '-march=rv32imc' '-mabi=ilp32'

junningwu@llvm-vm:~/workspace/pulp/test-progs/hello_haawking$ $PULP_RISCV_GCC_TOOLCHAIN/bin/riscv32-unknown-elf-objdump -D hello_haawking > hello_haawking.asm
junningwu@llvm-vm:~/workspace/pulp/test-progs/hello_haawking$ ../../pulp-tc/bin/riscv32-unknown-elf-gcc --static -march=RV32IMXpulpv1 -S -c hello_haawking.c
junningwu@llvm-vm:~/workspace/pulp/test-progs/hello_haawking$ cat hello_haawking.s 
	.file	"hello_haawking.c"
	.option nopic
	.text
	.align	2
	.globl	main
	.type	main, @function
main:
	add	sp,sp,-32
	sw	s0,28(sp)
	add	s0,sp,32
	sw	zero,-24(s0)
	sw	zero,-20(s0)
	j	.L2
.L3:
	lw	a4,-24(s0)
	lw	a5,-20(s0)
	p.mul	a4,a4,a5
	lw	a5,-24(s0)
	add	a5,a4,a5
	sw	a5,-24(s0)
	lw	a5,-20(s0)
	add	a5,a5,1
	sw	a5,-20(s0)
.L2:
	lw	a4,-20(s0)
	li	a5,9
	ble	a4,a5,.L3
	li	a5,0
	mv	a0,a5
	lw	s0,28(sp)
	add	sp,sp,32
	jr	ra
	.size	main, .-main
	.ident	"GCC: (GNU) 7.1.1 20170509"
```
直接写c，可以看到编译成p.mul指令。另外一种方式检验工具链最新增指令的支持，就是内嵌汇编，例如：
```
junningwu@llvm-vm:~/workspace/pulp/test-progs/test_p.mul$ cat test_p_mul.c 
/*
 * Copyright (c) 2019 Beijing Haawking Technology Co.,Ltd
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions are
 * met: redistributions of source code must retain the above copyright
 * notice, this list of conditions and the following disclaimer;
 * redistributions in binary form must reproduce the above copyright
 * notice, this list of conditions and the following disclaimer in the
 * documentation and/or other materials provided with the distribution;
 * neither the name of the copyright holders nor the names of its
 * contributors may be used to endorse or promote products derived from
 * this software without specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
 * "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
 * LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
 * A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
 * OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
 * LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
 * DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
 * THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
 * (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
 * OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 *
 * Authors: Junning Wu
 * Email: junning.wu@mail.haawking.com
 */

#include <stdio.h>

int main(){
  int a,b,c;
  a = 5;
  b = 2;
  asm volatile
  (
    "p.mac   %[z], %[x], %[y]\n\t"
    : [z] "=r" (c)
    : [x] "r" (a), [y] "r" (b)
  ) ; 
 
  if ( c != 10 ){
     printf("\nHAAWKING TEST: FAILED\n");
     return -1;
  }
  
  printf("\nHAAWKING TEST: PASSED\n");

  return 0;
}

junningwu@llvm-vm:~/workspace/pulp/test-progs/test_p.mul$ ../../pulp-tc/bin/riscv32-unknown-elf-gcc --static -march=RV32IMXpulpv1 -S -c test_p_mul.c
junningwu@llvm-vm:~/workspace/pulp/test-progs/test_p.mul$ cat test_p_mul.s 
	.file	"test_p_mul.c"
	.option nopic
	.section	.rodata
	.align	2
.LC0:
	.string	"\nHAAWKING TEST: FAILED"
	.align	2
.LC1:
	.string	"\nHAAWKING TEST: PASSED"
	.text
	.align	2
	.globl	main
	.type	main, @function
main:
	add	sp,sp,-32
	sw	ra,28(sp)
	sw	s0,24(sp)
	add	s0,sp,32
	li	a5,5
	sw	a5,-20(s0)
	li	a5,2
	sw	a5,-24(s0)
	lw	a5,-20(s0)
	lw	a4,-24(s0)
 #APP
# 38 "test_p_mul.c" 1
	p.mac   a5, a5, a4
	
# 0 "" 2
 #NO_APP
	sw	a5,-28(s0)
	lw	a4,-28(s0)
	li	a5,10
	beq	a4,a5,.L2
	lui	a5,%hi(.LC0)
	addi	a0,a5,%lo(.LC0)
	call	puts
	li	a5,-1
	j	.L3
.L2:
	lui	a5,%hi(.LC1)
	addi	a0,a5,%lo(.LC1)
	call	puts
	li	a5,0
.L3:
	mv	a0,a5
	lw	ra,28(sp)
	lw	s0,24(sp)
	add	sp,sp,32
	jr	ra
	.size	main, .-main
	.ident	"GCC: (GNU) 7.1.1 20170509"
```
## 安装Pulp-builder
首先下载pulp-builder的代码，根据说明修改branch。
```
junningwu@llvm-vm:~/workspace/pulp$ git clone https://github.com/pulp-platform/pulp-builder.git
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ git submodule update --init --recursive
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ git checkout 0e51ae60d66f4ec326582d63a9fcd40ed2a70e15
M	gvsoc
M	plptest
M	pmsis_api
M	pulp-configs
M	pulp-rt
M	pulp-rules
M	pulp-tools
M	runner
Note: checking out '0e51ae60d66f4ec326582d63a9fcd40ed2a70e15'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at 0e51ae6... Updated modules

junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ source configs/pulpissimo.sh
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ ./scripts/clean
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ ./scripts/update-runtime
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ ./scripts/build-runtime
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ source sdk-setup.sh
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ source configs/rtl.sh
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ cd ..
```
```
junningwu@llvm-vm:~/workspace/pulp/pulpissimo$ ./generate-scripts
```
### 安装VP平台GVSoC
编译安装GVSoC的VP平台，
```
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ source configs/pulpissimo.sh
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ ./scripts/build-gvsoc
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ source sdk-setup.sh
junningwu@llvm-vm:~/workspace/pulp/pulp-builder$ source configs/gvsoc.sh
```
完成之后，应该具有下列环境变量：
```
junningwu@llvm-vm:~/workspace/pulp/pulp-rt-examples/hello$ env | grep "PULP"
PULP_CURRENT_CONFIG=pulpissimo@config_file=chips/pulpissimo/pulpissimo.json
PULP_RISCV_GCC_TOOLCHAIN=/home/llvm/workspace/pulp/pulp-tc
PULP_CURRENT_CONFIG_ARGS=platform=gvsoc
PULP_SDK_WS_INSTALL=/home/llvm/workspace/pulp/pulp-builder/install/ws
PULP_SDK_HOME=/home/llvm/workspace/pulp/pulp-builder
PULP_SDK_INSTALL=/home/llvm/workspace/pulp/pulp-builder/install
PULP_CONFIGS_PATH=/home/llvm/workspace/pulp/pulp-builder/install/ws/configs
```
当前的SDK应该可以正常编译程序，但是执行的时候，还是存在一些问题。
```
junningwu@llvm-vm:~/workspace/pulp/pulp-rt-examples/hello$ make all
plpflags gen --input=pulpissimo@config_file=chips/pulpissimo/pulpissimo.json  --config=platform=gvsoc --output-dir=/home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo --makefile=/home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/config.mk    --app=test 
plpconf --input=pulpissimo@config_file=chips/pulpissimo/pulpissimo.json  --config=platform=gvsoc --output=/home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/config.json 
/home/llvm/workspace/pulp/pulp-tc/bin/riscv32-unknown-elf-gcc  -march=rv32imfcxpulpv2 -mfdiv -D__riscv__ -O3 -g  -fdata-sections -ffunction-sections -I/home/llvm/workspace/pulp/pulp-builder/install/include/io -I/home/llvm/workspace/pulp/pulp-builder/install/include -include /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/fc_config.h    -MMD -MP -c test.c -o /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/test/fc/test.o
/home/llvm/workspace/pulp/pulp-tc/bin/riscv32-unknown-elf-gcc  -march=rv32imfcxpulpv2 -mfdiv -D__riscv__ -O3 -g  -fdata-sections -ffunction-sections -I/home/llvm/workspace/pulp/pulp-builder/install/include/io -I/home/llvm/workspace/pulp/pulp-builder/install/include -include /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/fc_config.h    -MMD -MP -c /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/rt_conf.c -o /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/test/fc//home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/rt_conf.o
/home/llvm/workspace/pulp/pulp-tc/bin/riscv32-unknown-elf-gcc  -march=rv32imfcxpulpv2 -mfdiv -D__riscv__ -O3 -g  -fdata-sections -ffunction-sections -I/home/llvm/workspace/pulp/pulp-builder/install/include/io -I/home/llvm/workspace/pulp/pulp-builder/install/include -include /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/fc_config.h    -MMD -MP -c /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/rt_pad_conf.c -o /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/test/fc//home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/rt_pad_conf.o
mkdir -p `dirname /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/test/test`
/home/llvm/workspace/pulp/pulp-tc/bin/riscv32-unknown-elf-gcc -march=rv32imfcxpulpv2 -mfdiv -D__riscv__ -MMD -MP -o /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/test/test /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/test/fc/test.o /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/test/fc//home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/rt_conf.o /home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/test/fc//home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/rt_pad_conf.o  -nostartfiles -nostdlib -Wl,--gc-sections -L/home/llvm/workspace/pulp/pulp-builder/install/rules -Tpulpissimo/link.ld -L/home/llvm/workspace/pulp/pulp-builder/install/lib/pulpissimo -L/home/llvm/workspace/pulp/pulp-builder/install/lib/pulpissimo/pulpissimo -lrt -lrtio -lrt -lgcc
pulp-run --config-file=pulpissimo@config_file=chips/pulpissimo/pulpissimo.json  --config-opt=platform=gvsoc --dir=/home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo --binary=test/test prepare
Traceback (most recent call last):
  File "/home/llvm/workspace/pulp/pulp-builder/install/ws/bin/pulp-run", line 40, in <module>
    retval = runner.run()
  File "/home/llvm/workspace/pulp/pulp-builder/install/ws/python/plp_runner.py", line 232, in run
    retval = platform.handleCommands()
  File "/home/llvm/workspace/pulp/pulp-builder/install/ws/python/plp_platform.py", line 64, in handleCommands
    if self.execCommand(command) != 0: return 1
  File "/home/llvm/workspace/pulp/pulp-builder/install/ws/python/plp_platform.py", line 70, in execCommand
    return getattr(self, cmd)()
  File "/home/llvm/workspace/pulp/pulp-builder/install/ws/python/vp_runner.py", line 1277, in prepare
    encrypt=encrypted, aesKey=aes_key, aesIv=aes_iv):
TypeError: genFlashImage() got an unexpected keyword argument 'raw_fs'
/home/llvm/workspace/pulp/pulp-rt-examples/hello/build/pulpissimo/__rules.mk:118: recipe for target 'prepare' failed
make: *** [prepare] Error 1

```

## 安装pulp-sdk
在Ubuntu 16.04虚拟机中工作。 

```
junningwu@llvm-vm:~/workspace/pulp$ sudo apt install git python3-pip python-pip gawk texinfo libgmp-dev libmpfr-dev libmpc-dev swig3.0 libjpeg-dev lsb-core doxygen python-sphinx sox graphicsmagick-libmagick-dev-compat libsdl2-dev libswitch-perl libftdi1-dev cmake scons libsndfile1-dev
junningwu@llvm-vm:~/workspace/pulp$ sudo pip3 install artifactory twisted prettytable sqlalchemy pyelftools openpyxl xlsxwriter pyyaml numpy configparser pyvcd
junningwu@llvm-vm:~/workspace/pulp$ sudo pip2 install configparser
```
（各种尝试后，决定放弃治疗了。。。。。）
```
junningwu@llvm-vm:~/workspace/pulp$ git clone https://github.com/pulp-platform/pulp-sdk
junningwu@llvm-vm:~/workspace/pulp/pulp-sdk$ git submodule update --init --recursive
junningwu@llvm-vm:~/workspace/pulp/pulp-sdk$ source configs/pulpissimo.sh
junningwu@llvm-vm:~/workspace/pulp/pulp-sdk$ source configs/platform-rtl.sh
```