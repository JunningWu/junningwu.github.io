---
layout: post
title: 关于GCC编译器intrinsic函数的一些东东
date: 2019-12-02
categories: Tech
tags: [Eaasy,Tech]
description: 接触过比较多的intrinsic函数是TI的dsp所提供的库函数，为什么全世界都在用TI的DSP，就是因为他们针对不同的应用领域，提供了执行效率较高，用户编程友好的库函数，而这些大多数的库函数，都是采用intrinsic实现的。甚至对于一些加速器，比如Turbo码编译码器、FFT加速器等，使用的时候就跟调一个普通的函数一样，一个简单的函数调用就可以。
---

## 前言

在多媒体，图形和信号处理中，速度至关重要。有时，程序员会使用汇编语言来使机器获得最后的性能提升。 GCC在汇编语言和标准C语言之间提供了一个中间点，它可以为您提供更高的速度和处理器功能，而无需完全采用汇编语言：编译器内在函数intrinsic。
```
// TMS320C6400+, C6740, and C6600 C/C++ Compiler Intrinsics 
// ADDSUB
long long _addsub (int src1, int src2);  //Performs an addition and subtraction in parallel.
```
编译器内在函数intrinsic（有时称为“内联”）类似于常见的库函数，只是它们内置于编译器中。
它们可能比常规库函数更快（编译器对它们了解更多，因此可以更好地进行优化），或者处理的输入范围比库函数小。
内在函数还公开了处理器所特有的一些功能，因此也可以将它们用作标准C语言和汇编语言之间的中介。
这使我们能够使用类似于程序集的功能，但仍可以让编译器处理诸如类型检查，寄存器分配，指令调度和调用堆栈维护之类的细节。
一些内在程序是可移植的，而其他则不是，有些是局限在特定处理器的。可以在GCC信息页面和包含文件中找到可移植的和目标特定的内在函数的列表。
例如，[X86 Built-in Functions](https://gcc.gnu.org/onlinedocs/gcc-4.9.2/gcc/X86-Built-in-Functions.html)和[ARM NEON Intrinsics](https://gcc.gnu.org/onlinedocs/gcc-4.8.0/gcc/ARM-NEON-Intrinsics.html)以及[TI C6X Built-in Functions](https://gcc.gnu.org/onlinedocs/gcc-4.8.0/gcc/TI-C6X-Built_002din-Functions.html#TI-C6X-Built_002din-Functions)。

## 到底该不该使用Intrinsic函数（X86的一些讨论）
[12.5.3 Compiler Intrinsics, Power and Performance in Enterprise Systems, 2015](https://www.sciencedirect.com/topics/computer-science/compiler-intrinsics)
编译器内在函数是编译器提供的内置函数，它们与特定指令共享一对一或多对一关系。
这允许使用高级编程构造来编写特定指令，并使开发人员不必担心调用约定，寄存器分配和指令调度。
利用编译器内在函数的另一个优点是，与独立或内联汇编不同，编译器可以查看正在发生的情况并执行进一步的优化。
不幸的是，使用GCC编译内在函数可能会有些烦人。这是由于某些指令集只有在CFLAGS中显式启用时才能由编译器生成。
但是，如果在CFLAGS中启用了指令集，则可以在任何地方生成指令集，也就是说，不能保证所有指令都将受到CPUID检查的保护。
例如，尝试在不使用-mavx2编译器标志的情况下编译Intel AVX2编译器内部函数将导致编译失败。
为了绕过此问题，应将内部函数隔离到单独的文件中。这些文件只能包含根据CPUID的结果调度的功能。这是确保所有指令集扩展在运行时正确调度的唯一方法。
每个指令集扩展通常都有其自己的头文件。但是，由于指令集是相互构建的，因此仅应直接包含主要的顶级头文件。所有x86内部函数的主要头文件都是x86intrin.h，通常位于/usr/lib/gcc下的include目录中。
可以通过查看《英特尔软件开发人员手册》中指令参考下的相关指令文档来确定与特定指令相对应的内在函数。支持相应内部函数的每条指令都有一个名为“英特尔C/C ++编译器内部等效项”的部分，其中列出了内部签名。
支持较大的SIMD寄存器的内部函数会添加新的变量类型，以表示较大的宽度寄存器。例如，__m128代表通用的128位SSE寄存器，而__m128i代表存储打包整数的128位SSE寄存器。

当然，其实对于某一个特定领域的应用，例如TI，使用Intrinsic函数就比较常见和普通了。

## 怎么样添加自己的Intrinsic函数
那么怎么才能添加自己的Intrinsic函数呢？作为一个初学者，我Google一下，也没找到关于修改gcc添加Intrinsic函数的帖子，只找到了一个扩展llvm的官方教程。[Extending LLVM: Adding instructions, intrinsics, types, etc.](https://llvm.org/docs/ExtendingLLVM.html)。
