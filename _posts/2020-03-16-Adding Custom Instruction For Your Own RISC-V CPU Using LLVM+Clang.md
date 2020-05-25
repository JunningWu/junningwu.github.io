---
layout: post
title: Adding Custom Instruction For Your Own RISC-V CPU Using LLVM+Clang
date: 2020-03-16
categories: Tech
tags: [Essay,Tech]
description: 本篇主要记录一下初学使用LLVM的后端完成自定义指令集的汇编和反汇编。由于一些情况下，自定义指令集的指令格式与标准指令集不一样，通过修改Gnu Binutils的方式，不能套用已有的函数定义和声明，复杂度会比较高，博主经过数次尝试，依然不能完成这项任务，所以就转向了LLVM。目前来看，一切运行的还不错，就先做个阶段性的工作总结记录一下。
---

我在之前的帖子中，介绍了通过修改GNU Binutils的方式来完成自定义指令集的汇编和反汇编，以及几个不同的修改层次，修改和调试工作随着层次越高也变得越来越复杂。我们（Haawking.com）通过这种方式，完成了一些自定义指令的汇编和反汇编，但是对于另外一些，则无法进行下去，只能通过insn的形式来实现。

![llvm-clang-architecture](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/llvm-clang/llvm-clang-architecture.png)

最近，我们决定采用LLVM+Clang的方式来完成这项工作，第一步先通过修改LLVM的后端，来实现自定义指令集的汇编和反汇编。LLVM是一个很大的工程，但是又不像GCC那样你无从下手去修改，如果说GCC是X86指令集架构，那么LLVM更像是RISC-V指令集架构，一个模块化的编译器框架，你可以修改任何一部分，然后借助于其他模块完成整个编译器的工作。比较有意思的是，LLVM的创始人，Chris Lattner [维基百科](https://en.wikipedia.org/wiki/Chris_Lattner) ，履历相当精彩，上学期间做的一个项目叫做Low-level Virtual Machine，毕业以后加入了苹果，在乔布斯手下工作，后来加入了特斯拉，半年以后从特斯拉离职后加入了谷歌，今年（2020年）加入了RISC-V初创公司SiFive。

其实我个人知道LLVM这个编译器，是大概在2013年前后，当时磊哥还在所里工作，负责编译器和工具链，然后还有勇爷。只是一直没有机会深入研究，这次由于公司业务需要，硬着头皮顶上去，发现还真是一个好项目。据了解，现在大多数的AI芯片创业公司都在招募LLVM编译器人才，就是希望能够借助于编译器来优化自家软件和硬件协同工作的性能。而Lattner在Google的那两年，一直在为TPU工作，即使在特斯拉的时间比较短，也是为Autopilot项目服务。

其实，目前针对于RISC-V的LLVM编译器，还只是一个半成品，如果我在网上公开可以获得信息是准确的话（其实现在我编译程序的时候也得到了印证），程序的头和链接部分，还依赖于RISC-V GNU Toolchain，也就是说，如果你想让LLVM编译器可以编译出来自家的RISC-V处理器可以执行的可执行代码，那么你还需要准备一个RISC-V GNU Toolchain。不过，怎么说呢，至少你可以不用采用insn了。

![sifive-bruce-hoult-aug-18](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/llvm-clang/sifive-bruce-hoult-aug-18.png)

对于指令mulsrN来说，完成两个数相乘并对结果进行移位，由于需要三个源操作数（两个源寄存器和一个立即数），目前RISC-V标准指令格式中要实现的话，就需要拆分已有含义的指令位，就像pulp采用将高7位拆分成funct2和uimm5两个部分。

```
mulsrN   a2, a1, a0, 2

mulsrN rd rs1 rs2 uimm5 31..30=3 14..12=1 6..2=0x02 1..0=3
```

通过LLVM支持自定义指令集，需要修改MC Layer的td文件，具体修改如下：

```
//~/llvm-project/llvm/lib/Target/RISCV

//修改RISCVInstrFormats.td，增加自定义指令格式的类RVInstRI
class RVInstRI<bits<2> funct2, bits<3> funct3, RISCVOpcode opcode, dag outs, dag ins, string opcodestr, string argstr>
    : RVInst<outs, ins, opcodestr, argstr, [], InstFormatRI> {
  bits<5> rs2;
  bits<5> rs1;
  bits<5> uimm5;
  bits<5> rd;

  let Inst{31-30} = funct2;
  let Inst{29-25} = uimm5;
  let Inst{24-20} = rs2;
  let Inst{19-15} = rs1;
  let Inst{14-12} = funct3;
  let Inst{11-7} = rd;
  let Opcode = opcode.Value;
}

//修改RISCVInstrInfo.td，添加自定义格式mulsrN
let hasSideEffects = 0, mayLoad = 0, mayStore = 0 in
def MULSRN : RVInstRI<0b11, 0b001, OPC_HAAWKING, (outs GPR:$rd), (ins GPR:$rs1, GPR:$rs2, uimm5:$shift),
 "mulsri", "$rd, $rs1, $rs2, $shift">, Sched<[]> {
 bits<5> shift;
 bits<5> rs1;
 bits<5> rs2;
 bits<5> rd;

 let Inst{31-30} = 0b11;
 let Inst{29-25} = shift;
 let Inst{24-20} = rs2;
 let Inst{19-15} = rs1;
 let Inst{14-12} = 0b001;
 let Inst{11-7} = rd;
 let Opcode = OPC_HAAWKING.Value;
}

```

然后就是编译LLVM了，上面的代码应该没有问题，这里为了减少编译时间，采用ninja，而且编译选项也是只选择了RISCV目标系统。

```
mkdir build
cd build
cmake -G Ninja -DCMAKE_INSTALL_PREFIX=/home/llvm/workspace/llvm/llvm-project/llvm_install -DCMAKE_BUILD_TYPE="Release" -DLLVM_ENABLE_PROJECTS="clang;lld;libc" -DLLVM_TARGETS_TO_BUILD="RISCV" ../llvm
cmake --build .
ninja install
```

编译完成以后，在安装目录下就可以看到编译好的工具链了，如下所示：

```
llvm@llvm-vm:~/workspace/llvm$ ls llvm-project/llvm_install/
bin  include  lib  libexec  riscv64-unknown-elf  share
llvm@llvm-vm:~/workspace/llvm$ ls llvm-project/llvm_install/bin/
bugpoint              clang-import-test      ld64.lld         llvm-cat         llvm-dis                llvm-lib         llvm-nm          llvm-reduce      llvm-xray
c-index-test          clang-offload-bundler  ld.lld           llvm-cfi-verify  llvm-dlltool            llvm-link        llvm-objcopy     llvm-rtdyld      obj2yaml
clang                 clang-offload-wrapper  llc              llvm-config      llvm-dwarfdump          llvm-lipo        llvm-objdump     llvm-size        opt
clang++               clang-refactor         lld              llvm-cov         llvm-dwp                llvm-lto         llvm-opt-report  llvm-split       sancov
clang-11              clang-rename           lld-link         llvm-c-test      llvm-elfabi             llvm-lto2        llvm-pdbutil     llvm-stress      sanstats
clang-check           clang-scan-deps        lli              llvm-cvtres      llvm-exegesis           llvm-mc          llvm-profdata    llvm-strings     scan-build
clang-cl              diagtool               llvm-addr2line   llvm-cxxdump     llvm-extract            llvm-mca         llvm-ranlib      llvm-strip       scan-view
clang-cpp             dsymutil               llvm-ar          llvm-cxxfilt     llvm-ifs                llvm-ml          llvm-rc          llvm-symbolizer  verify-uselistorder
clang-extdef-mapping  git-clang-format       llvm-as          llvm-cxxmap      llvm-install-name-tool  llvm-modextract  llvm-readelf     llvm-tblgen      wasm-ld
clang-format          hmaptool               llvm-bcanalyzer  llvm-diff        llvm-jitlink            llvm-mt          llvm-readobj     llvm-undname     yaml2obj
```

同时，为了能够正常使用Newlib库的一些函数，也需要单独编译针对目标系统的Newlib库。具体操作如下：

```
% git clone https://github.com/riscv/riscv-newlib.git
% cd riscv-newlib 
% mkdir build
% cd build
% ../riscv-newlib/configure --target=riscv32-unknown-elf --prefix=$INSTALL_DIR
% make all
% make install
```

一般这一步也不会出现什么错误，基本上过一会就编译完成了。下面就是使用编译好的LLVM+Clang来编译我们的程序了。这里的hello world程序如下：

```
int main(){
  int a,b,c;
  a = 5;
  b = 2;
  asm volatile
  (
    "mulsrN   %[z], %[x], %[y], 2\n\t"
    : [z] "=r" (c)
    : [x] "r" (a), [y] "r" (b)
  ) ; 
  if ( c != 28 ){
    print(0);
    return -1;
  }
  print(1);
  return 0;
}
```

编译命令如下：

```
RISCV_GCC_OPTS ?= -mcmodel=medany -static -O3 -std=gnu99 -fno-common -fno-builtin -march=rv32im -mabi=ilp32 -DMB_ADDR=0x80FFFFC --target=riscv64-unknown-elf --sysroot=/home/llvm/workspace/riscv/riscv-tc-20200220/bin/riscv64-unknown-elf --gcc-toolchain=/home/llvm/workspace/riscv/riscv-tc-20200220
RISCV_LINK_OPTS ?= -static -nostdlib -nostartfiles -lm -lgcc -T /home/llvm/workspace/HRV_IDE/common/test.ld
newlib_dir := /home/llvm/workspace/llvm/llvm-project/llvm_install/riscv64-unknown-elf/include
src_dir := $(WORK_DIR)/src/$(PROJ)
incs  += -I$(WORK_DIR)/env -I$(WORK_DIR)/common -I$(src_dir) -I$(newlib_dir)
src := $(wildcard $(src_dir)/*.c) $(wildcard $(WORK_DIR)/common/*.c) $(wildcard $(WORK_DIR)/common/*.S)

$(RISCV_clang) $(incs) $(RISCV_GCC_OPTS) -o $(WORK_DIR)/build/$@/$@ $(src) $(RISCV_LINK_OPTS) --verbose
$(RISCV_OBJDUMP) -d $(WORK_DIR)/build/$@/$@ > $(WORK_DIR)/build/$@/$@.S
```

采用GCC OBJDUMP进行反汇编后的代码如下：

```
04001048 <main>:
 4001048:	ff010113          	addi	sp,sp,-16
 400104c:	00112623          	sw	ra,12(sp)
 4001050:	00812423          	sw	s0,8(sp)
 4001054:	00500513          	li	a0,5
 4001058:	00200593          	li	a1,2
 400105c:	c4b5150b          	0xc4b5150b
 4001060:	01c00593          	li	a1,28
 4001064:	02b51263          	bne	a0,a1,4001088 <main+0x40>
 4001068:	08100537          	lui	a0,0x8100
 400106c:	ffc50513          	addi	a0,a0,-4 # 80ffffc <__global_pointer$+0xff7fc>
 4001070:	00100593          	li	a1,1
 4001074:	418000ef          	jal	ra,400148c <print>
 4001078:	00000413          	li	s0,0
 400107c:	deadc537          	lui	a0,0xdeadc
 4001080:	eef50593          	addi	a1,a0,-273 # deadbeef <__global_pointer$+0xd6adb6ef>
 4001084:	00c0006f          	j	4001090 <main+0x48>
 4001088:	00000593          	li	a1,0
 400108c:	fff00413          	li	s0,-1
 4001090:	08100537          	lui	a0,0x8100
 4001094:	ffc50513          	addi	a0,a0,-4 # 80ffffc <__global_pointer$+0xff7fc>
 4001098:	3f4000ef          	jal	ra,400148c <print>
 400109c:	00040513          	mv	a0,s0
 40010a0:	00812403          	lw	s0,8(sp)
 40010a4:	00c12083          	lw	ra,12(sp)
 40010a8:	01010113          	addi	sp,sp,16
 40010ac:	00008067          	ret
```

采用LLVM OBJDUMP进行反汇编后的代码如下：

```
04001048 main:
 4001048: 13 01 01 ff                  	addi	sp, sp, -16
 400104c: 23 26 11 00                  	sw	ra, 12(sp)
 4001050: 23 24 81 00                  	sw	s0, 8(sp)
 4001054: 13 05 50 00                  	addi	a0, zero, 5
 4001058: 93 05 20 00                  	addi	a1, zero, 2
 400105c: 0b 15 b5 c4                  	mulsri	a0, a0, a1, 2
 4001060: 93 05 c0 01                  	addi	a1, zero, 28
 4001064: 63 12 b5 02                  	bne	a0, a1, 36
 4001068: 37 05 10 08                  	lui	a0, 33024
 400106c: 13 05 c5 ff                  	addi	a0, a0, -4
 4001070: 93 05 10 00                  	addi	a1, zero, 1
 4001074: ef 00 80 41                  	jal	1048
 4001078: 13 04 00 00                  	mv	s0, zero
 400107c: 37 c5 ad de                  	lui	a0, 912092
 4001080: 93 05 f5 ee                  	addi	a1, a0, -273
 4001084: 6f 00 c0 00                  	j	12
 4001088: 93 05 00 00                  	mv	a1, zero
 400108c: 13 04 f0 ff                  	addi	s0, zero, -1
 4001090: 37 05 10 08                  	lui	a0, 33024
 4001094: 13 05 c5 ff                  	addi	a0, a0, -4
 4001098: ef 00 40 3f                  	jal	1012
 400109c: 13 05 04 00                  	mv	a0, s0
 40010a0: 03 24 81 00                  	lw	s0, 8(sp)
 40010a4: 83 20 c1 00                  	lw	ra, 12(sp)
 40010a8: 13 01 01 01                  	addi	sp, sp, 16
 40010ac: 67 80 00 00                  	ret
```

上面程序地址为0x400105c的那条指令，也就是0xc4b5150b，就是我们程序中的mulsrN指令码。

其实，走到这一步，LLVM编译器也只是按照我们的修改完成了指令码的生成，但是为了完整性，还需要添加指令的test程序，能够对指令生成进行正确性测试。这部分就等到后面更新帖子的时候再添加。

这里面还有一部分工作是在摸索中才明白的，比如添加sysroot选项，就是因为不添加的话，会调用host系统的ld程序，肯定不能交叉编译出来RISC-V的程序代码。还有，多亏了都春霞的指导，不然也不可能这么快就知道只用修改llvm的后端，就可以让clang识别内嵌的汇编指令，这样也避免了直接编写汇编程序的尴尬。

<Junning Wu，20200316，Beijing>