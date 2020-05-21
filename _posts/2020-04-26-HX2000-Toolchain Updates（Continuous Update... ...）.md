---
layout: post
title: HX2000-Toolchain Updates（Continuous Update... ...）
date: 2020-04-26
categories: Tech
tags: [Essay,Tech]
description: 由于科研文档库于今日403（FxxK，▄█▀█●），现在将HX2000-Toolchain的更新移到GitHub上面来。
---

# HX2000-Toolchain

## RISC-V和LLVM原始源代码版本信息

- riscv-gnu-toolchain：2a2def26e9ab4ce24ef4266ad5d2f3fd8bd20169
- llvm-project：llvm-project-llvmorg-10.0.0
- riscv-newlib：f289cef6be67da67b2d97a47d6576fa7e6b4c858

每次更新一版Toolchain，相应源代码也会进行备份，暂定为按照日期。

## HX2000-Toolchain OS版本

### Ubuntu （16.04）

- riscv-tc-gcc：riscv-tc-20200317-rv32-multilib.tar.gz
- riscv-tc-llvm：dsc_llvm_20200430.tar.gz
- IDE-LLVM：IDE_LLVM-20200430.zip

1. **V20200326**:支持《2020_DSC_HX2802x__v0.6》，并根据ADDSR和SUBSR调整，修改LLVM工具链支持ADDSRI和SUBSRI

2. **V20200430**：支持《2020_DSC_HX2802x__v1.0》。


### CentOS（8.1）

- riscv-tc-gcc： riscv-tc-gcc-centos8-20200414.tar.xz
- riscv-tc-llvm： riscv-tc-llvm-centos8-20200430.zip
- IDE-LLVM： IDE_LLVM-20200430.zip

1. **V20200414**：支持《2020_DSC_HX2802x__v0.6》。

2. **V20200417**: 支持《2020_DSC_HX2802x_指令提案_v0.7》。

3. **V20200430**：支持《2020_DSC_HX2802x__v1.0》。

### Windows（win7、win10）

- IDE-LLVM： IDE_DSC_LLVM_Win-Rel-20200424.zip
- IDE-LLVM： IDE_DSC_LLVM_Win-Rel-20200430.zip
- IDE-LLVM： IDE_DSC_LLVM_Win-Rel-20200518.zip

1. **V20200424**：支持《2020_DSC_HX2802x__v0.9》。

2. **V20200430**：支持《2020_DSC_HX2802x__v1.0》。

2. **V20200518**：修复一些小问题。

## 如何使用

1.分别下载riscv-tc-llvm、riscv-tc-gcc和IDE压缩包并解压。对于Windows版本的IDE，可以跳过步骤2和步骤3，解压之后可以直接运行compile.bat，路径已经设置完成，且GCC和LLVM已经包括在内。

2.修改IDE根目录下compile文件，\$ENV{"PATH"} = \$ENV{"PATH"}.":<解压路径>/riscv-tc-gcc/bin";\$ENV{"PATH"} = \$ENV{"PATH"}.":<解压路径>/riscv-tc-llvm/bin";$gcc_dir = "<解压路径>/riscv-tc-gcc";

3.修改IDE/MK目录下Makefile文件，newlib_incdir ?= <解压路径>/riscv-tc-llvm/riscv32-unknown-elf/include；newlib_libdir ?= <解压路径>/riscv-tc-llvm/riscv32-unknown-elf/lib

4.查看IDE/src目录下的示例程序test_newinst，里面包括部分自定义指令的内联汇编代码，确认无误后，回到IDE根目录，执行./compile test_newinst。如果没有报错，可以在IDE/build目录下看到示例程序test_newinst的编译结果以及反汇编程序，其中.S为gcc objdump反汇编结果（自定义指令显示为32比特16进制数），.ASM为llvm-objdump反汇编结果（显示自定义指令助记符）。

5.如不需要查看编译详细过程，可以将IDE/MK/Makefile中，clang的编译选项--verbose删除。

6.如需要联合编译C源代码程序和ASM源代码程序，可以通过修改 IDE/MK/Makefile中src变量，增加.s文件。默认Makefile只支持C源代码程序，Makefile_ASM只支持ASM源代码程序。 

7.如果首次编译，提示clang或者其他命令不存在，请查看PATH路径是否设置成功。

8.如果centos虚拟机存在无法与宿主机通信的情况，可以通过设置共享文件夹的形式传输文件。共享文件夹在虚拟机的路径为/mnt/hgfs/。

9.如果采用步骤8之后，没有在/mnt/hgfs目录下看到共享文件，可以执行

//ubuntu
```
# sudo vmware-hgfsclient
# sudo vmhgfs-fuse .host:/share /share -o allow_other -o uid=1001 -o gid=1001 
```
//centos
```
# su
# vmware-hgfsclient
# vmhgfs-fuse .host:/share /share -o allow_other -o uid=1001 -o gid=1001 
```

如不能解决，请联系junning.wu@mail.haawking.com。

10.如果使用过程中提示缺少Windows的动态链接库，如mscvp140.dll或者vcruntime.dll，可以通过下载之后，放在windows\System32目录下。如果是在内网调试，可以在\lirw\software\DSC_IDE_dll目录下找到这两个文件，放在windows\System32目录下即可。




## CentOS虚拟机使用说明

为了方便使用CentOS版本IDE调试和减少系统重复安装，提供了CentOS8.1的VMware镜像文件，用户名为haawking（已经设为管理员用户），解压密码为hk2019，root密码为ct110。

请使用VMware workstation pro 12以上版本打开该镜像文件。

RTL仿真服务器上面已经安装了VMware程序，可以直接使用。


## 已知问题

### 1.与服务器OS不匹配

RTL仿真服务器操作系统为Redhat6.5和Redhat6.10，LLVM编译工具暂不支持。可以使用Windows版的IDE或者使用CentOS的虚拟机。

### 2.连续寄存器分配暂不支持

 对于 DMAC、LQP、LDP 、SDP等《2020_DSC_HX2802x__v0.6》中源/目的寄存器为连续多个的指令，对于 MUL32、MUL32U、MUL32SU等《2020_DSC_HX2802x__v0.7》中源/目的寄存器为连续两个的指令，当前版本IDE支持A0固定寄存器的连续分配，即A0A1，A0A1A2A3，内嵌汇编程序的编写，参考IDE中示例程序。

### 3.LLVM编译工具不支持rv32imc

 目前工具链支持rv32i/ilp32、rv32im/ilp32、rv32iac/ilp32、rv32imac/ilp32 、rv32imafc/ilp32f 、rv64imac/lp64 、rv64imafdc/lp64d；不支持rv32imc，暂时可以用rv32imac替代。

### 4.遇到DIV、REM指令执行会出错
 
 DSC不支持除法操作，但是当前版本编译器支持RV32imc指令集，因此会编译出div/rem等指令，执行的时候会出错。
 