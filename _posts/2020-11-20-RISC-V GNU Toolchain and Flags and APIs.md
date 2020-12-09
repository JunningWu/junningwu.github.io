---
layout: post
title: RISC-V GNU Toolchain and Flags and APIs
date: 2020-11-20
updated: 2020-11-20
categories: Tech
tags: [Essay,Tech]
description: RISC-V GNU Toolchain中包括两个比较关键的参数，会影响指令的生成，那就是-march和-mabi，那这两个参数到底怎么设置呢？这篇博文就把搜集到的一些资料整理出来，供参考。
---

## GNU网站上面介绍的RISC-V Options

这里的说明[3.19.42 RISC-V Options](https://gcc.gnu.org/onlinedocs/gcc/RISC-V-Options.html) ，会随着gcc版本的升级而更新，这里以gcc 10.2版本为例说明。



```
-mabi=ABI-string
Specify integer and floating-point calling convention. 
ABI-string contains two parts: the size of integer types and the registers used for floating-point types. 
For example ‘-march=rv64ifd -mabi=lp64d’ means that ‘long’ and pointers are 64-bit (implicitly defining ‘int’ to be 32-bit), and that floating-point values up to 64 bits wide are passed in F registers. 
Contrast this with ‘-march=rv64ifd -mabi=lp64f’, which still allows the compiler to generate code that uses the F and D extensions but only allows floating-point values up to 32 bits long to be passed in registers;
 or ‘-march=rv64ifd -mabi=lp64’, in which no floating-point arguments will be passed in registers.

The default for this argument is system dependent, users who want a specific calling convention should specify one explicitly. 
The valid calling conventions are: ‘ilp32’, ‘ilp32f’, ‘ilp32d’, ‘lp64’, ‘lp64f’, and ‘lp64d’. Some calling conventions are impossible to implement on some ISAs: 
for example, ‘-march=rv32if -mabi=ilp32d’ is invalid because the ABI requires 64-bit values be passed in F registers, but F registers are only 32 bits wide.
 There is also the ‘ilp32e’ ABI that can only be used with the ‘rv32e’ architecture. This ABI is not well specified at present, and is subject to change.
```

```
-march=ISA-string
Generate code for given RISC-V ISA (e.g. ‘rv64im’). ISA strings must be lower-case. Examples include ‘rv64i’, ‘rv32g’, ‘rv32e’, and ‘rv32imaf’.

When -march= is not specified, use the setting from -mcpu.

If both -march and -mcpu= are not specified, the default for this argument is system dependent, users who want a specific architecture extensions should specify one explicitly.
```

## 具体含义

尽管在[RISC-V GNU Compiler Toolchain](https://github.com/riscv/riscv-gnu-toolchain) 的说明中，有一个简单的介绍，但是可能很多还是不知道具体指的是什么。

```
Supported ABIs are ilp32 (32-bit soft-float), ilp32d (32-bit hard-float), ilp32f (32-bit with single-precision in registers and double in memory, niche use only), lp64 lp64f lp64d (same but with 64-bit long and pointers).
```

而在[RISC-V ABIs](https://wiki.gentoo.org/wiki/RISC-V_ABIs) Gentoo的wiki网站上面，却有一个较为详细的说明。

```
RISC-V has two integer ABIs and three floating-point ABIs, which can essentially be combined at will. 
Code generation is controlled by the -mabi argument to compiler calls, which concatenates the integer and floating point ABI name.

Example: -mabi=ilp32d

The choice of ABI places requirements on the instruction set supported by the hardware and emitted by the compiler.
```

### Integer ABIs
ilp32
- int, long, pointers are 32bit
- long long is 64bit
- char is 8bit
- short is 16bit
ilp32 is currently only supported for 32-bit targets.

lp64
- int is 32bit
- long and pointers are 64bit
- long long is 64bit
- char is 8bit
- short is 16bit
lp64 is only supported for 64-bit targets.

### Floating Point ABIs
"" (empty string)
- No floating point arguments are passed in registers.
- No requirements on instruction set / hardware.
f
- 32bit and smaller floating point types are passed in registers.
- Requires F type floating point registers and instructions.
d
- 64bit and smaller floating point types are passed in registers.
- Requires D type floating point registers and instructions.


## 如何针对性地生成Multilib

采用默认的配置，使能```-enable-multilib```，编译出来的可能不是我们需要的组合，需要手动进行配置。

可以通过修改```gcc/config/riscv/t-elf-multilib```文件，添加自己需要的组合，比如，我个人添加了rv32imfc/ilp32f（因为我不需要A指令集），则再次编译之后，可以生成对应的库。

```
MULTILIB_OPTIONS = march=rv32i/march=rv32ic/march=rv32im/march=rv32imc/march=rv32iac/march=rv32imac/march=rv32imafc/march=rv32imfc/march=rv32imafdc/march=rv32gc/march=rv64imac/march=rv64imafdc/march=rv64gc mabi=ilp32/mabi=ilp32f/mabi=lp64/mabi=lp64d
MULTILIB_DIRNAMES = rv32i \
rv32ic \
rv32im \
rv32imc \
rv32iac \
rv32imac \
rv32imafc \
rv32imfc \
rv32imafdc \
rv32gc \
rv64imac \
rv64imafdc \
rv64gc ilp32 \
ilp32f \
lp64 \
lp64d
MULTILIB_REQUIRED = march=rv32i/mabi=ilp32 \
march=rv32im/mabi=ilp32 \
march=rv32iac/mabi=ilp32 \
march=rv32imac/mabi=ilp32 \
march=rv32imc/mabi=ilp32 \
march=rv32imafc/mabi=ilp32f \
march=rv32imfc/mabi=ilp32f \
march=rv64imac/mabi=lp64 \
march=rv64imafdc/mabi=lp64d
```

编译的结果如下：

```
ab@haawking-pc1 MINGW64 ~/work/riscv-gnu-toolchain-58c9d86
$ ../HX2000-Toolchain/riscv-tc-gcc-1123/bin/riscv64-unknown-elf-gcc.exe --print-multi-lib
.;
rv32i/ilp32;@march=rv32i@mabi=ilp32
rv32im/ilp32;@march=rv32im@mabi=ilp32
rv32imc/ilp32;@march=rv32imc@mabi=ilp32
rv32iac/ilp32;@march=rv32iac@mabi=ilp32
rv32imac/ilp32;@march=rv32imac@mabi=ilp32
rv32imafc/ilp32f;@march=rv32imafc@mabi=ilp32f
rv32imfc/ilp32f;@march=rv32imfc@mabi=ilp32f
rv64imac/lp64;@march=rv64imac@mabi=lp64
rv64imafdc/lp64d;@march=rv64imafdc@mabi=lp64d

```

## Library介绍

### The libunwind project

The primary goal of this project is to define a portable and efficient C programming interface (API) to determine the call-chain of a program. 

The API additionally provides the means to manipulate the preserved (callee-saved) state of each call-frame and to resume execution at any point in the call-chain (non-local goto). 

The API supports both local (same-process) and remote (across-process) operation. As such, the API is useful in a number of applications. Some examples include:

- **exception handling**
The libunwind API makes it trivial to implement the stack-manipulation aspects of exception handling.
- **debuggers**
The libunwind API makes it trivial for debuggers to generate the call-chain (backtrace) of the threads in a running program.
- **introspection**
It is often useful for a running thread to determine its call-chain. For example, this is useful to display error messages (to show how the error came about) and for performance monitoring/analysis.
- **efficient setjmp()**
With libunwind, it is possible to implement an extremely efficient version of setjmp(). Effectively, the only context that needs to be saved consists of the stack-pointer(s).

### Debugio Library


### CXX-noexcept

### libc Library

The GNU C Library, commonly known as glibc, is the GNU Project's implementation of the C standard library. 

Despite its name, it now also directly supports C++ (and, indirectly, other programming languages). 

It was started in the early 1990s by the Free Software Foundation (FSF) for their GNU operating system.

Released under the GNU Lesser General Public License, glibc is free software. The GNU C Library project provides the core libraries for the GNU system and GNU/Linux systems, 

as well as many other systems that use Linux as the kernel. These libraries provide critical APIs including ISO C11, POSIX.1-2008, BSD, OS-specific APIs and more. 

These APIs include such foundational facilities as open, read, write, malloc, printf, getaddrinfo, dlopen, pthread_create, crypt, login, exit and more.

glibc is used in systems that run many different kernels and different hardware architectures. Its most common use is in systems using the Linux kernel on x86 hardware, 

however, officially supported hardware includes: 32-bit ARM and its newer 64-bit ISA (AArch64), C-SKY, DEC Alpha, IA-64, Motorola m68k, MicroBlaze, MIPS, Nios II, PA-RISC, PowerPC, RISC-V, 

s390, SPARC, and x86 (old versions support TILE). It officially supports the Hurd and Linux kernels. Additionally, there are heavily patched versions that run on the kernels of 

FreeBSD and NetBSD (from which Debian GNU/kFreeBSD and Debian GNU/NetBSD systems are built, respectively), as well as a forked-version of OpenSolaris. 

It is also used (in an edited form) and named libroot.so in BeOS and Haiku.

### libm Library


## GCC可以正常编译程序，LLVM则无法编译连接

### Clang的三个编译选项

- --gcc-toolchain=<value> Use the gcc toolchain at the given directory
- --target=<value>        Generate code for the given target
- --sysroot=<path>        When you have extracted your cross-compiler from a zip file into a directory, you have to use --sysroot=<path>. The path is the root directory where you have unpacked your file, and Clang will look for the directories bin, lib, include in there.
- -mcmodel=medany         Equivalent to -mcmodel=medium, compatible with RISC-V gcc.
- -mcmodel=medlow         Equivalent to -mcmodel=small, compatible with RISC-V gcc.
- -msave-restore          Enable using library calls for save and restore
- -msmall-data-limit=<value> Put global and static data smaller than the limit into a special section
- -mtune=<value>          Only supported on X86 and RISC-V. Otherwise accepted for compatibility with GCC.
- -print-target-triple    Print the normalized target triple
- -print-targets          Print the registered targets
- -T <script>             Specify <script> as linker script



### 目前Clang编译不支持rv32imfc+ilp32f的组合，需是rv32imafc+ilp32f

Depending on the options you choose it might try to find that specific multilib. So if you will go wild, your compiler might not have that multilib present and use the default 64bit one. So if you will get strange errors target emulation 'elf64-littleriscv' does not match 'elf32-littleriscv' then it could mean you selected combination of features that your compiler does not have included the specific multilib.

通过修改```clang/lib/Driver/ToolChains/Gnu.cpp```中findRISCVBareMetalMultilibs，添加rv32imfc+ilp32f和rv32imc+ilp32的组合，重新编译LLVM，则就可以支持。

从文件的注释中可以看出，目前版本的LLVM，并未支持类似的组合，尽管有该组合的产品存在。

```
clang version 11.0.0 (https://github.com/llvm/llvm-project.git 176249bd6732a8044d457092ed932768724a6f06)
Target: riscv32-unknown-unknown-elf
Thread model: posix
InstalledDir: D:\work\Haawking-IDE-Eclipse-CDT\Haawking-IDE-Eclipse-CDT.win32.x86_64\haawking-tools\compiler\Haawking-RISCV-LLVM\bin
Found candidate GCC installation: D:/work/Haawking-IDE-Eclipse-CDT/Haawking-IDE-Eclipse-CDT.win32.x86_64/haawking-tools/compiler/riscv-tc-gcc/lib/gcc/riscv64-unknown-elf\10.2.0
Selected GCC installation: D:/work/Haawking-IDE-Eclipse-CDT/Haawking-IDE-Eclipse-CDT.win32.x86_64/haawking-tools/compiler/riscv-tc-gcc/lib/gcc/riscv64-unknown-elf/10.2.0
Candidate multilib: rv32i/ilp32;@march=rv32i@mabi=ilp32
Candidate multilib: rv32im/ilp32;@march=rv32im@mabi=ilp32
Candidate multilib: rv32iac/ilp32;@march=rv32iac@mabi=ilp32
Candidate multilib: rv32imac/ilp32;@march=rv32imac@mabi=ilp32
Candidate multilib: rv32imc/ilp32;@march=rv32imc@mabi=ilp32
Candidate multilib: rv32imafc/ilp32f;@march=rv32imafc@mabi=ilp32f
Candidate multilib: rv32imfc/ilp32f;@march=rv32imfc@mabi=ilp32f
Candidate multilib: rv64imac/lp64;@march=rv64imac@mabi=lp64
Candidate multilib: rv64imafdc/lp64d;@march=rv64imafdc@mabi=lp64d
Selected multilib: rv32imc/ilp32;@march=rv32imc@mabi=ilp32
```


（2020-10-19，希格玛公寓，北京）