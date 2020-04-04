---
layout: post
title: Supporting Hidden Consecutive Op Registers in LLVM Backend for RISC-V
date: 2020-04-04
categories: Tech
tags: [Essay,Tech]
description: 本篇博文主要记录一下通过修改LLVM RISC-V的后端代码，以支持DSC指令集中需要连续两个或者四个源、目的寄存器的情况。由于指令编码不允许将所有寄存器显式指出，因此在编码中只给出起始寄存器编号，隐藏的一个或者三个已被占用的寄存器，就不能被其他操作数使用，否则将会产生计算错误。
---

通过修改LLVM的RA是可以完成这个工作的，但是由于本人能力有限，且RA修改难度较大，因此选择了一个折衷的方式，就是通过生命几个新的寄存器组，来约束指令rs和rd寄存器的选择，希望能够通过这种方式来实现一个初级版本。这里，对于两个连续寄存器，则为A0A1，如果是连续四个寄存器，则为A0A1A2A3。

在RISC-V的LLVM后端代码中/lib/Target/RISCV/RISCVRegisterInfo.td，已经定义了GPR寄存器组，也就是32个通用寄存器。

```
// The order of registers represents the preferred allocation sequence.
// Registers are listed in the order caller-save, callee-save, specials.
def GPR : RegisterClass<"RISCV", [XLenVT], 32, (add
    (sequence "X%u", 10, 17),
    (sequence "X%u", 5, 7),
    (sequence "X%u", 28, 31),
    (sequence "X%u", 8, 9),
    (sequence "X%u", 18, 27),
    (sequence "X%u", 0, 4)
  )> {
  let RegInfos = RegInfoByHwMode<
      [RV32,              RV64,              DefaultMode],
      [RegInfo<32,32,32>, RegInfo<64,64,64>, RegInfo<32,32,32>]>;
}
```

1. 根据需要，在文件/lib/Target/RISCV/RISCVRegisterInfo.td中我们重新定义三个寄存器组，GPRA0，GPRNOA0A1，GPRNOA0A1A2A3。

```
//Haawking Custom Instructions
def GPRA0 : RegisterClass<"RISCV", [XLenVT], 32, (add X10)> {
  let RegInfos = RegInfoByHwMode<
      [RV32,              RV64,              DefaultMode],
      [RegInfo<32,32,32>, RegInfo<64,64,64>, RegInfo<32,32,32>]>;
}

//Haawking Custom Instructions
def GPRNOA0A1 : RegisterClass<"RISCV", [XLenVT], 32, (add
    (sequence "X%u", 12, 17),
    (sequence "X%u", 5, 7),
    (sequence "X%u", 28, 31),
    (sequence "X%u", 8, 9),
    (sequence "X%u", 18, 27),
    (sequence "X%u", 0, 4)
  )> {
  let RegInfos = RegInfoByHwMode<
      [RV32,              RV64,              DefaultMode],
      [RegInfo<32,32,32>, RegInfo<64,64,64>, RegInfo<32,32,32>]>;
}

//Haawking Custom Instructions
def GPRNOA0A1A2A3 : RegisterClass<"RISCV", [XLenVT], 32, (add
    (sequence "X%u", 14, 17),
    (sequence "X%u", 5, 7),
    (sequence "X%u", 28, 31),
    (sequence "X%u", 8, 9),
    (sequence "X%u", 18, 27),
    (sequence "X%u", 0, 4)
  )> {
  let RegInfos = RegInfoByHwMode<
      [RV32,              RV64,              DefaultMode],
      [RegInfo<32,32,32>, RegInfo<64,64,64>, RegInfo<32,32,32>]>;
}
```

2. 接着，在文件/lib/Target/RISCV/RISCVInstrInfo.td中修改我们修改一下指令LQP的定义，将rd寄存器和rs1、rs2寄存器的组别指定为新的寄存器组。

```
//LQP
let hasSideEffects = 0, mayLoad = 1, mayStore = 0 in
def LQP : RVInstRI<0b11, 0b000, OPC_HX_CUS0, 
                  (outs GPRA0:$rd), (ins GPRNOA0A1A2A3:$rs1, GPRNOA0A1A2A3:$rs2, simm5:$shift),
                  "lqp", "$rd, $rs1, $rs2, $shift">, Sched<[]> {
 bits<5> shift;
 bits<5> rs1;
 bits<5> rs2;
 bits<5> rd;

 let Inst{31-30} = 0b11;
 let Inst{29-25} = shift;
 let Inst{24-20} = rs2;
 let Inst{19-15} = rs1;
 let Inst{14-12} = 0b000;
 let Inst{11-7} = rd;
 let Opcode = OPC_HX_CUS0.Value;
}
```

3. 第三步，在文件/lib/Target/RISCV/Disassembler/RISCVDisassembler.cpp中我们修改一下，加入新寄存器组的DecodeStatus类。

```
//Haawking Custom Instructions
static DecodeStatus DecodeGPRA0RegisterClass(MCInst &Inst, uint64_t RegNo,
                                               uint64_t Address,
                                               const void *Decoder) {
  if (RegNo != 10) {
    return MCDisassembler::Fail;
  }

  return DecodeGPRRegisterClass(Inst, RegNo, Address, Decoder);
}

//Haawking Custom Instructions
static DecodeStatus DecodeGPRNOA0A1A2A3RegisterClass(MCInst &Inst, uint64_t RegNo,
                                               uint64_t Address,
                                               const void *Decoder) {
  if (RegNo == 10 || RegNo == 11 || RegNo == 12 || RegNo == 13) {
    return MCDisassembler::Fail;
  }

  return DecodeGPRRegisterClass(Inst, RegNo, Address, Decoder);
}

static DecodeStatus DecodeGPRNOA0A1RegisterClass(MCInst &Inst, uint64_t RegNo,
                                               uint64_t Address,
                                               const void *Decoder) {
  if (RegNo == 10 || RegNo == 11) {
    return MCDisassembler::Fail;
  }

  return DecodeGPRRegisterClass(Inst, RegNo, Address, Decoder);
}
```

4. 重新编译一下LLVM+Clang，没有报错。然后，编写一个测试的应用程序，代码段如下：

```
  asm volatile
  (
    "lqp   %[z], %[x], %[y], 4\n\t"
    : [z] "=r" (c)
    : [x] "r" (a), [y] "r" (b)
  ) ;
```

使用Clang编译的时候，会报错，看着像是生成指令码的时候还是按照之前的规则生成的，只不过在检查的时候出错（报错应该是与上面DecodeStatus有关）

```
clang -I./env -I./common -I./src/test_newinst -I/home/llvm/workspace/llvm/llvm-project/llvm_install/riscv32-unknown-elf/include -mcmodel=medany -static -std=gnu99 -fno-common -fno-builtin-printf -march=rv32imac -mabi=ilp32 -DMB_ADDR=0x11ffC -O3 --target=riscv32-unknown-elf --sysroot=/home/llvm/workspace/riscv/riscv-tc-20200316/bin/riscv32-unknown-elf --gcc-toolchain=/home/llvm/workspace/riscv/riscv-tc-20200316 -o ./build/test_newinst/test_newinst ./src/test_newinst/main.c ./common/syscalls.c ./common/dev.c ./common/crt.S -static  -nostartfiles -lm -lgcc -T ./common/test.ld

./src/test_newinst/main.c:117:5: error: invalid operand for instruction
    "lqp   %[z], %[x], %[y], 4\n\t"
    ^
	
<inline asm>:1:8: note: instantiated into assembly here
        lqp   a2, a0, a1, 4
```

///////////////我是分割线/////////////2020-04-04//////////////////


