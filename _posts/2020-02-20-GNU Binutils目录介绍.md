---
layout: post
title: GNU Binutils目录框架及简介
date: 2020-02-20
categories: Tech
tags: [Tech]
description: 最近因为工作需要，对于自定义的指令进行支持，感觉写.word或者.insn对于单条指令的验证和测试还算ok，但是如果指令数量较多，则就不太合适，还是需要工具能够进行汇编生成可执行代码的。而这部分工作需要修改的就是GNU Binutils。
---

GNU Binutils源代码的大部分位于下列这几个目录中。binutils的某些组件是库，可在内部以及其他项目中使用。例如，BFD库用于GNU GDB调试器。这些库具有自己的顶级目录。GNU Binutils的目录框架如下：

 ** +binutils **
 - include  
 - bfd  
 - opcodes   
 - cpu  
 - binutils  
 - gas  
 - ld  
 - gprof  
 - gold  
 - elfcpp

头文件，提供横跨主要组件的信息。例如，主模拟器接口标头位于此处（remote-sim.h），因为它将GDB（在gdb目录中）链接到模拟器（在sim目录中）。特定于组件的其他标头位于该组件的目录中。

二进制文件描述符库。该库包含用于处理特定二进制文件格式的代码，例如ELF，COFF，SREC等。如果必须识别新的目标文件类型，则应在此处添加支持它的代码。

操作码库。其中包含有关如何组装和拆卸指令的信息。

名为CGEN的实用程序的源文件。该工具可用于自动为opcodes库以及GDB使用的SIM卡模拟器生成目标特定的源文件。

尽管它的名字是binutils，但不是主binutils目录。而是所有binutils工具的目录，这些工具没有自己的顶级源目录。这包括objcopy，objdump和readelf等工具。

GNU汇编器。目标特定的汇编代码保存在config子目录

GNU链接器。目标特定的链接器文件保存在子目录中。

GNU探查器。该程序没有任何目标特定的代码

新的GNU链接器。这是一个新的链接器，用于替换LD。目前，它仍在开发中。

Elfcpp是一个用于读取和写入ELF信息的C ++库。当前仅由GOLD链接器使用。