---
layout: post
title: HX2000-Toolchain Updates（Continuous Update... ...）
date: 2020-04-26
updated: 2020-06-12
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
- IDE-LLVM： IDE_DSC_LLVM_Win-Rel-20200523.zip
- IDE-LLVM： IDE_DSC_LLVM_Win-Rel-20200602.zip
- IDE-LLVM： IDE_DSC_LLVM_Win-Rel-20200612.zip
- IDE-LLVM： IDE_DSC_LLVM_Win-Rel-20200616.zip

1. **V20200424**：支持《2020_DSC_HX2802x__v0.9》。

2. **V20200430**：支持《2020_DSC_HX2802x__v1.0》。

3. **V20200518**：修复一些小问题。

4. **V20200523**：支持double、float、整型等数据的乘除法操作。

5. **V20200602**：使用静态库编译工具链，不依赖与操作系统的某些DLL库。

6. **V20200612**：大更新版本：调整工程文件目录结构，ld文件和库文件以及Driver都统一放置于src目录下；修复编译器RPTI指令立即数错误；调整连续寄存器相关指令从X10到X12，涉及到32位乘法指令和LQP/LDP/SDP等。

7. **V20200616**：支持128bit Flash；新增dret指令支持；添加汇编程序编译环境。

## 如何使用

### 1.
分别下载riscv-tc-llvm、riscv-tc-gcc和IDE压缩包并解压。对于Windows版本的IDE，可以跳过步骤2和步骤3，解压之后可以直接运行compile.bat，路径已经设置完成，且GCC和LLVM已经包括在内。

### 2.
修改IDE根目录下compile文件，\$ENV{"PATH"} = \$ENV{"PATH"}.":<解压路径>/riscv-tc-gcc/bin";\$ENV{"PATH"} = \$ENV{"PATH"}.":<解压路径>/riscv-tc-llvm/bin";$gcc_dir = "<解压路径>/riscv-tc-gcc";

### 3.
修改IDE/MK目录下Makefile文件，newlib_incdir ?= <解压路径>/riscv-tc-llvm/riscv32-unknown-elf/include；newlib_libdir ?= <解压路径>/riscv-tc-llvm/riscv32-unknown-elf/lib

### 4.
查看IDE/src目录下的示例程序test_newinst，里面包括部分自定义指令的内联汇编代码，确认无误后，回到IDE根目录，执行./compile test_newinst。如果没有报错，可以在IDE/build目录下看到示例程序test_newinst的编译结果以及反汇编程序，其中.S为gcc objdump反汇编结果（自定义指令显示为32比特16进制数），.ASM为llvm-objdump反汇编结果（显示自定义指令助记符）。

### 5.
如不需要查看编译详细过程，可以将IDE/MK/Makefile中，clang的编译选项--verbose删除。

### 6.
如需要联合编译C源代码程序和ASM源代码程序，可以通过修改 IDE/MK/Makefile中src变量，增加.s文件。默认Makefile只支持C源代码程序，Makefile_ASM只支持ASM源代码程序。 

### 7.
如果首次编译，提示clang或者其他命令不存在，请查看PATH路径是否设置成功。

### 8.
如果centos虚拟机存在无法与宿主机通信的情况，可以通过设置共享文件夹的形式传输文件。共享文件夹在虚拟机的路径为/mnt/hgfs/。

### 9.
如果采用步骤8之后，没有在/mnt/hgfs目录下看到共享文件，可以执行

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

### 10.
如果使用过程中提示缺少Windows的动态链接库，如mscvp140.dll或者vcruntime.dll，可以通过下载之后，放在windows\System32目录下。如果是在内网调试，可以在\lirw\software\DSC_IDE_dll目录下找到这两个文件，放在windows\System32目录下即可。

通过静态编译的方式，版本新于20200602的IDE应该不会存在依赖库找不到的问题。

### 11. 
新增示例程序，C程序hello_haawking和ASM汇编程序test_dret，新建工程可拷贝复制之后进行修改。如程序需在Flash执行，可参考svgen_dp，进行使能等的配置。


## CentOS虚拟机使用说明

为了方便使用CentOS版本IDE调试和减少系统重复安装，提供了CentOS8.1的VMware镜像文件，用户名为haawking（已经设为管理员用户），解压密码为hk2019，root密码为ct110。

请使用VMware workstation pro 12以上版本打开该镜像文件。

RTL仿真服务器上面已经安装了VMware程序，可以直接使用。


## 已知问题

### 1.与服务器OS不匹配

RTL仿真服务器操作系统为Redhat6.5和Redhat6.10，LLVM编译工具暂不支持。可以使用Windows版的IDE或者使用CentOS的虚拟机。

### 2.连续寄存器分配暂不支持

 对于 DMAC、LQP、LDP 、SDP等《2020_DSC_HX2802x__v0.6》中源/目的寄存器为连续多个的指令，对于 MUL32、MUL32U、MUL32SU等《2020_DSC_HX2802x__v0.7》中源/目的寄存器为连续两个的指令，当前版本IDE支持A2固定寄存器的连续分配，即A2A3，A2A3A4A5，内嵌汇编程序的编写，参考IDE中示例程序。

### 3.LLVM编译工具不支持rv32imc

 目前工具链支持rv32i/ilp32、rv32im/ilp32、rv32iac/ilp32、rv32imac/ilp32 、rv32imafc/ilp32f 、rv64imac/lp64 、rv64imafdc/lp64d；不支持rv32imc，暂时可以用rv32imac替代。

  【20200523】更新：将M指令集中的乘法指令提了出来，目前编译选项为rv32ic，可以编译出乘法指令。

### 4.遇到DIV、REM指令执行会出错
 
 DSC不支持除法操作，但是当前版本编译器支持RV32imc指令集，因此会编译出div/rem等指令，执行的时候会出错。【这一问题在20200523版本中已经得到解决】
 