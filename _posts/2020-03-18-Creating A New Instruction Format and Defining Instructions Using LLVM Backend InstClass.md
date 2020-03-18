---
layout: post
title: Creating A New Instruction Format and Defining Instructions Using LLVM Backend InstClass
date: 2020-03-18
categories: Tech
tags: [Essay,Tech]
description: 在上一篇帖子中，我通过添加新的指令格式实现了MULSRI的汇编和反汇编，这次由于同类型格式的指令较多，LLVM是支持生命诚新的类，来方便例化指令的，但是在实际操作的时候，遇到了问题，编译出来的指令码对不上，同时，无法进行反汇编。
---


对于指令mulsrN来说，完成两个数相乘并对结果进行移位，由于需要三个源操作数（两个源寄存器和一个立即数），目前RISC-V标准指令格式中要实现的话，就需要拆分已有含义的指令位，就像pulp采用将高7位拆分成funct2和uimm5两个部分。可以正确汇编和反汇编的LLVM修改如下。

```
mulsrN   a2, a1, a0, 2
mulsrN rd rs1 rs2 uimm5 31..30=3 14..12=1 6..2=0x02 1..0=3
```

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

现在有了若干条同样格式的指令，其实在RISC-V LLVM已有的后端实现中，对于诸如add，sub，sll，slt等指令，就是采用这样的方式，重新定义一个ALU_rr类，简化了指令的描述，只用传递funct7和funct3就行。

```
def ADD  : ALU_rr<0b0000000, 0b000, "add">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def SUB  : ALU_rr<0b0100000, 0b000, "sub">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def SLL  : ALU_rr<0b0000000, 0b001, "sll">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def SLT  : ALU_rr<0b0000000, 0b010, "slt">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def SLTU : ALU_rr<0b0000000, 0b011, "sltu">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def XOR  : ALU_rr<0b0000000, 0b100, "xor">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def SRL  : ALU_rr<0b0000000, 0b101, "srl">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def SRA  : ALU_rr<0b0100000, 0b101, "sra">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def OR   : ALU_rr<0b0000000, 0b110, "or">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def AND  : ALU_rr<0b0000000, 0b111, "and">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
```

其实，对于与MULSRI具有同样格式的指令，我也希望能够定义一个类，比如ALU_ri，这样就可以很方便的例化指令，而不用把类中每个字段都重复写一遍。

```
let hasSideEffects = 0, mayLoad = 0, mayStore = 0 in
class ALU_ri<bits<2> funct2, bits<3> funct3, string opcodestr>
    : RVInstRI<funct2, funct3, OPC_HX_CUS0, (outs GPR:$rd), (ins GPR:$rs1, GPR:$rs2, uimm5:$shift),
              opcodestr, "$rd, $rs1, $rs2, $shift">;
```

例化指令的时候这样写：

```
def MULSRI   : ALU_HX_ri<0b11, 0b001, "mulsri">, Sched<[]>;
```

然而，这样写，clang是可以正确编译出来可执行代码的，但是指令码是错误的，比如，对于指令mulsri	a0, a0, a1, 2，正确的指令码应该是0xc4b5150b，但是通过gcc-objdump进行反汇编后的指令码是0xd4b5150b。而且，通过llvm-objdump进行反汇编，会出现如下的错误。

```
Stack dump:
0.	Program arguments: /home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump -d /home/llvm/workspace/HRV_IDE/build/test_mulsri/test_mulsri 
 #0 0x00000000005f3cfa llvm::sys::PrintStackTrace(llvm::raw_ostream&) (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x5f3cfa)
 #1 0x00000000005f1f1c llvm::sys::RunSignalHandlers() (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x5f1f1c)
 #2 0x00000000005f2083 SignalHandler(int) (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x5f2083)
 #3 0x00007f531cec5390 __restore_rt (/lib/x86_64-linux-gnu/libpthread.so.0+0x11390)
 #4 0x0000000000535ea0 llvm::MCExpr::print(llvm::raw_ostream&, llvm::MCAsmInfo const*, bool) const (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x535ea0)
 #5 0x000000000049ee9e llvm::RISCVInstPrinter::printInstruction(llvm::MCInst const*, unsigned long, llvm::MCSubtargetInfo const&, llvm::raw_ostream&) (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x49ee9e)
 #6 0x000000000049f001 llvm::RISCVInstPrinter::printInst(llvm::MCInst const*, unsigned long, llvm::StringRef, llvm::MCSubtargetInfo const&, llvm::raw_ostream&) (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x49f001)
 #7 0x000000000043a3c6 llvm::(anonymous namespace)::PrettyPrinter::printInst(llvm::MCInstPrinter&, llvm::MCInst const*, llvm::ArrayRef<unsigned char>, llvm::object::SectionedAddress, llvm::raw_ostream&, llvm::StringRef, llvm::MCSubtargetInfo const&, llvm::(anonymous namespace)::SourcePrinter*, llvm::StringRef, std::vector<llvm::object::RelocationRef, std::allocator<llvm::object::RelocationRef> >*) (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x43a3c6)
 #8 0x000000000044546f llvm::disassembleObject(llvm::Target const*, llvm::object::ObjectFile const*, llvm::MCContext&, llvm::MCDisassembler*, llvm::MCDisassembler*, llvm::MCInstrAnalysis const*, llvm::MCInstPrinter*, llvm::MCSubtargetInfo const*, llvm::MCSubtargetInfo const*, llvm::(anonymous namespace)::PrettyPrinter&, llvm::(anonymous namespace)::SourcePrinter&, bool) (.constprop.864) (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x44546f)
 #9 0x0000000000448b47 llvm::disassembleObject(llvm::object::ObjectFile const*, bool) (.constprop.862) (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x448b47)
#10 0x0000000000449dac llvm::dumpObject(llvm::object::ObjectFile*, llvm::object::Archive const*, llvm::object::Archive::Child const*) (.constprop.854) (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x449dac)
#11 0x00000000004153f1 main (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x4153f1)
#12 0x00007f531be26830 __libc_start_main /build/glibc-LK5gWL/glibc-2.23/csu/../csu/libc-start.c:325:0
#13 0x000000000042d419 _start (/home/llvm/workspace/llvm/llvm-project/llvm_install/bin/llvm-objdump+0x42d419)
Segmentation fault (core dumped)
```



<Junning Wu，20200318，Beijing>