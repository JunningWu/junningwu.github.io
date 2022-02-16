---
layout: post
title: Porting Haawking LLVM-GCC Toolchain from LLVM11.0.0 to LLVM13.0.1
date: 2022-02-16
updated: 2022-02-16
categories: Tech
tags: [Essay,Tech]
description: 大概从2020年过年复工开始，我就在做一些LLVM移植的工作，通过修改LLVM后端，支持昊芯自定义指令集，当然是从汇编层面支持自定义指令，使用者需要编写汇编代码。最近LLVM13.0.1终于在千呼万唤中发布了，有很多比较重要的升级，在这里感谢社区的付出。如果没有社区的付出，我一个外行人，也不可能在这么短的时间支持数十条自定义指令。也趁着这个版本更新的机会，把做的修改记录一下，方便后面再继续开展工作。
---

## 全部修改文件列表
在支持昊芯自定义指令的过程中，一共修改了7个文件，主要根据指令编码格式，增加新的编码定义。

在13.0.1版本中，LLVM的目录结构发生了变化，之前的Utils文件夹已经去掉了。

```
llvm/lib/Target/RISCV/AsmParser/RISCVAsmParser.cpp
llvm/lib/Target/RISCV/Disassembler/RISCVDisassembler.cpp
llvm/lib/Target/RISCV/MCTargetDesc/RISCVBaseInfo.h
llvm/lib/Target/RISCV/RISCVInstrFormats.td
llvm/lib/Target/RISCV/RISCVRegisterInfo.td
llvm/lib/Target/RISCV/RISCVInstrInfo.td
llvm/lib/Target/RISCV/RISCVISelLowing.cpp
clang/lib/Driver/ToolChains/Gnu.cpp
```


### RISCVAsmParser.cpp

在自定义指令中，相比较原版指令集，增加了两个立即数操作数，SImm5和UImm12，因此需要对操作数进行判断。不过，在LLVM13.0.1中，SImm5已经存在，所以相对来说，只新增了一个。

```
/* Haawking Custom Instructions By Junning Wu*/
  bool isUImm12() const {
    int64_t Imm;
    RISCVMCExpr::VariantKind VK = RISCVMCExpr::VK_RISCV_None;
    if (!isImm())
      return false;
    bool IsConstantImm = evaluateConstantImm(getImm(), Imm, VK);
    return IsConstantImm && isUInt<12>(Imm) && VK == RISCVMCExpr::VK_RISCV_None;
  }
```

### RISCVDisassembler.cpp

在自定义指令中，由于存在隐藏寄存器的使用，由于笔者能力有限，目前只能通过寄存器组的形式，强制实现这个功能，增加了NoA2A3A4A5、NoA2A3和A2寄存器组。

```
//Haawking Custom Instructions By Junning Wu
static DecodeStatus DecodeGPRA2RegisterClass(MCInst &Inst, uint64_t RegNo,
                                               uint64_t Address,
                                               const void *Decoder) {
  if (RegNo != 12) {
    return MCDisassembler::Fail;
  }

  return DecodeGPRRegisterClass(Inst, RegNo, Address, Decoder);
}
```

### RISCVBaseInfo.h

增加两个新的OperandType，其实这次只增加了一个。

```
OPERAND_UIMM12, /* Haawking Custom Instructions By Junning Wu*/
```


### RISCVInstrFormats.td

增加了InstFormatRI和两个RISCVOpcode类型。

```
def InstFormatRI     : InstFormat<18>;  //Haawking Custom Instructions By Junning Wu
def OPC_HX_CUS0   : RISCVOpcode<0b0001011>;  //Haawking Custom Instructions By Junning Wu
def OPC_HX_CUS1   : RISCVOpcode<0b0101011>;  //Haawking Custom Instructions By Junning Wu
```
以及RVInstRI类的定义。
```
//Haawking Custom Instructions By Junning Wu
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
```

### RISCVRegisterInfo.td

如前所述，定义三个寄存器组。
```
//Haawking Custom Instructions By Junning Wu
def GPRA2 : RegisterClass<"RISCV", [XLenVT], 32, (add X12)> {
  let RegInfos = RegInfoByHwMode<
      [RV32,              RV64,              DefaultMode],
      [RegInfo<32,32,32>, RegInfo<64,64,64>, RegInfo<32,32,32>]>;
}
```

### RISCVInstrInfo.td
首先，定义uimm12和simm5，
```
//Haawking Custom Instructions By Junning Wu
def uimm12 : Operand<XLenVT>, ImmLeaf<XLenVT, [{return isUInt<12>(Imm);}]> {
  let ParserMatchClass = UImmAsmOperand<12>;
  let DecoderMethod = "decodeUImmOperand<12>";
  let OperandType = "OPERAND_UIMM12";
  let OperandNamespace = "RISCVOp";
}
```
其次，定义三个昊芯专用的指令格式，ALU_HX_rr、ALU_HX_ri、ALU_HX_rr2等。

```
//Haawking Custom Instructions By Junning Wu
let hasSideEffects = 0, mayLoad = 0, mayStore = 0 in
class ALU_HX_rr<bits<7> funct7, bits<3> funct3, string opcodestr>
    : RVInstR<funct7, funct3, OPC_HX_CUS0, (outs GPR:$rd), (ins GPR:$rs1, GPR:$rs2),
              opcodestr, "$rd, $rs1, $rs2">;
```
最后，就是定义每一条指令，具体实现这里就不在赘述，可以参考源码。
```
//Haawking Custom Instructions By Junning Wu
def SADD   : ALU_HX_rr<0b0000000, 0b000, "sadd">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def SSUB   : ALU_HX_rr<0b0100000, 0b000, "ssub">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def MINN   : ALU_HX_rr<0b0000000, 0b010, "min">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
def MAXX   : ALU_HX_rr<0b0100001, 0b010, "max">, Sched<[WriteIALU, ReadIALU, ReadIALU]>;
```

### RISCVISelLowing.cpp

修改ISelLowing文件，主要是为了能够使用C嵌汇编的编程。主要增加对特殊寄存器组的识别。
```
asm volatile(
	"srl64li %[z],%[x],%[y],2 \n\t"
			: [z]"=r"(c)
			: [x]"r" (a),[y] "r" (b)
	);
```

### clang/lib/Driver/ToolChains/Gnu.cpp

通过修改findRISCVBareMetalMultilibs，添加rv32imfc+ilp32f和rv32imc+ilp32的组合，重新编译LLVM，则就可以支持。

```
// currently only support the set of multilibs like riscv-gnu-toolchain does.
  // TODO: support MULTILIB_REUSE
  constexpr RiscvMultilib RISCVMultilibSet[] = {
      {"rv32i", "ilp32"},     {"rv32im", "ilp32"},     {"rv32iac", "ilp32"},
      {"rv32imac", "ilp32"}, {"rv32imc", "ilp32"},  {"rv32imafc", "ilp32f"}, {"rv32imfc", "ilp32f"}, {"rv64imac", "lp64"},
```

最后，目前版本的编译器，能够完成自定义指令的编译和反汇编，并未添加测试，对于隐藏寄存器的支持，也只是能够在限定寄存器的情况下能用。这是因为寄存器分配的实现难度较高。

（2022-02-16，财智国际大厦，北京）
