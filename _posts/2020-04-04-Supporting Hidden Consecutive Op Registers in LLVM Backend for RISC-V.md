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

查看一下clang生成的LLVM IR，使用命令为clang -S -emit-llvm main.c

```
; Function Attrs: noinline nounwind optnone
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  store i32 5, i32* %2, align 4
  store i32 2, i32* %3, align 4
  %5 = load i32, i32* %2, align 4
  %6 = load i32, i32* %3, align 4
  %7 = call i32 asm sideeffect "lqp   $0, $1, $2, 4\0A\09", "=r,r,r"(i32 %5, i32 %6) #1, !srcloc !3
  store i32 %7, i32* %4, align 4
  ret i32 0
}

```

///////////////我是分割线/////////////2020-04-06//////////////////

通过查找错误信息“invalid operand for instruction”，我们发现在/lib/Target/RISCV/AsmParser/RISCVAsmParser.cpp中，存在下面的函数调用，函数RISCVAsmParser::MatchAndEmitInstruction调用函数MatchInstructionImpl，根据执行后的返回结果，判断Match_Success、Match_MissingFeature、Match_MnemonicFail、Match_InvalidOperand。而且通过增加调试信息，可以确定编译函数所报的错误，就是这段代码的作用。

其实这部分主要完成的工作是Instruction Alias Processing，指令别名处理，指令解析后，将进入MatchInstructionImpl函数。MatchInstructionImpl函数执行别名处理，然后进行实际匹配。别名处理是将相同指令的不同词汇形式规范化为一个表示形式的阶段。别名有几种不同的实现方式，它们的处理顺序如下（从最简单/最弱到最复杂/最强大）。通常，希望使用满足指令要求的第一种别名机制，因为这将允许更简洁的描述。

```
auto Result =
    MatchInstructionImpl(Operands, Inst, ErrorInfo, MissingFeatures,
                         MatchingInlineAsm);

... ...

case Match_InvalidOperand: {
    SMLoc ErrorLoc = IDLoc;
    if (ErrorInfo != ~0U) {
      if (ErrorInfo >= Operands.size())
        return Error(ErrorLoc, "too few operands for instruction");

      ErrorLoc = ((RISCVOperand &)*Operands[ErrorInfo]).getStartLoc();
      if (ErrorLoc == SMLoc())
        ErrorLoc = IDLoc;
    }
    return Error(ErrorLoc, "invalid operand for instruction");
  }
```

通过增加指令别名的方式，也没有解决出现的问题。

```
def : InstAlias<"lqp $rd, $rs1, $rs2, $shift", (LQP GPRA0:$rd, GPRNOA0A1A2A3:$rs1, GPRNOA0A1A2A3:$rs2, simm5:$shift)>;
```

通过查找文件build/lib/Target/RISCV/RISCVGenMCCodeEmitter.inc，找到如下代码，是函数uint64_t RISCVMCCodeEmitter::getBinaryCodeForInstr(const MCInst &MI, SmallVectorImpl<MCFixup> &Fixups, const MCSubtargetInfo &STI)的片段。是生成指令二进制码的代码段。其中，比较重要的函数是getMachineOpValue()。

```
case RISCV::ADDSRI:
case RISCV::LQP:
case RISCV::MAHSRI:
case RISCV::MALSRI:
case RISCV::MSHSRI:
case RISCV::MSLSRI:
case RISCV::MULSRI:
case RISCV::SUBSRI: {
      // op: rs2
      op = getMachineOpValue(MI, MI.getOperand(2), Fixups, STI);
      op &= UINT64_C(31);
      op <<= 20;
      Value |= op;
      // op: rs1
      op = getMachineOpValue(MI, MI.getOperand(1), Fixups, STI);
      op &= UINT64_C(31);
      op <<= 15;
      Value |= op;
      // op: rd
      op = getMachineOpValue(MI, MI.getOperand(0), Fixups, STI);
      op &= UINT64_C(31);
      op <<= 7;
      Value |= op;
      // op: shift
      op = getMachineOpValue(MI, MI.getOperand(3), Fixups, STI);
      op &= UINT64_C(31);
      op <<= 25;
      Value |= op;
      break;
    }
```

通过追踪，找到了函数getMachineOpValue()的定义，这个函数的功能应该是把输入的寄存器或者立即数，转换成数值形式，为了使得函数getBinaryCodeForInstr()能够根据二进制数将32位指令码填充完毕。

```
unsigned
RISCVMCCodeEmitter::getMachineOpValue(const MCInst &MI, const MCOperand &MO,
                                      SmallVectorImpl<MCFixup> &Fixups,
                                      const MCSubtargetInfo &STI) const {

  if (MO.isReg())
    return Ctx.getRegisterInfo()->getEncodingValue(MO.getReg());

  if (MO.isImm())
    return static_cast<unsigned>(MO.getImm());

  llvm_unreachable("Unhandled expression!");
  return 0;
}
```

重新再回来看看函数MatchInstructionImpl()，也没有发现跟问题相关的代码段或者逻辑。

```
unsigned RISCVAsmParser::
MatchInstructionImpl(const OperandVector &Operands,
                     MCInst &Inst,
                     uint64_t &ErrorInfo,
                     FeatureBitset &MissingFeatures,
                     bool matchingInlineAsm, unsigned VariantID) {
  // Eliminate obvious mismatches.
  if (Operands.size() > 6) {
    ErrorInfo = 6;
    return Match_InvalidOperand;
  }

  // Get the current feature set.
  const FeatureBitset &AvailableFeatures = getAvailableFeatures();

  // Get the instruction mnemonic, which is the first token.
  StringRef Mnemonic = ((RISCVOperand&)*Operands[0]).getToken();

  // Process all MnemonicAliases to remap the mnemonic.
  applyMnemonicAliases(Mnemonic, AvailableFeatures, VariantID);

  // Some state to try to produce better error messages.
  bool HadMatchOtherThanFeatures = false;
  bool HadMatchOtherThanPredicate = false;
  unsigned RetCode = Match_InvalidOperand;
  MissingFeatures.set();
  // Set ErrorInfo to the operand that mismatches if it is
  // wrong for all instances of the instruction.
  ErrorInfo = ~0ULL;
  // Find the appropriate table for this asm variant.
  const MatchEntry *Start, *End;
  switch (VariantID) {
  default: llvm_unreachable("invalid variant!");
  case 0: Start = std::begin(MatchTable0); End = std::end(MatchTable0); break;
  }
  // Search the table.
  auto MnemonicRange = std::equal_range(Start, End, Mnemonic, LessOpcode());

  DEBUG_WITH_TYPE("asm-matcher", dbgs() << "AsmMatcher: found " <<
  std::distance(MnemonicRange.first, MnemonicRange.second) << 
  " encodings with mnemonic '" << Mnemonic << "'\n");

  // Return a more specific error code if no mnemonics match.
  if (MnemonicRange.first == MnemonicRange.second)
    return Match_MnemonicFail;

  for (const MatchEntry *it = MnemonicRange.first, *ie = MnemonicRange.second;
       it != ie; ++it) {
    const FeatureBitset &RequiredFeatures = FeatureBitsets[it->RequiredFeaturesIdx];
    bool HasRequiredFeatures =
      (AvailableFeatures & RequiredFeatures) == RequiredFeatures;
    DEBUG_WITH_TYPE("asm-matcher", dbgs() << "Trying to match opcode "
                                          << MII.getName(it->Opcode) << "\n");
    // equal_range guarantees that instruction mnemonic matches.
    assert(Mnemonic == it->getMnemonic());
    bool OperandsValid = true;
    for (unsigned FormalIdx = 0, ActualIdx = 1; FormalIdx != 5; ++FormalIdx) {
      auto Formal = static_cast<MatchClassKind>(it->Classes[FormalIdx]);
      DEBUG_WITH_TYPE("asm-matcher",
                      dbgs() << "  Matching formal operand class " << getMatchClassName(Formal)
                             << " against actual operand at index " << ActualIdx);
      if (ActualIdx < Operands.size())
        DEBUG_WITH_TYPE("asm-matcher", dbgs() << " (";
                        Operands[ActualIdx]->print(dbgs()); dbgs() << "): ");
      else
        DEBUG_WITH_TYPE("asm-matcher", dbgs() << ": ");
      if (ActualIdx >= Operands.size()) {
        DEBUG_WITH_TYPE("asm-matcher", dbgs() << "actual operand index out of range ");
        OperandsValid = (Formal == InvalidMatchClass) || isSubclass(Formal, OptionalMatchClass);
        if (!OperandsValid) ErrorInfo = ActualIdx;
        break;
      }
      MCParsedAsmOperand &Actual = *Operands[ActualIdx];
      unsigned Diag = validateOperandClass(Actual, Formal);
      if (Diag == Match_Success) {
        DEBUG_WITH_TYPE("asm-matcher",
                        dbgs() << "match success using generic matcher\n");
        ++ActualIdx;
        continue;
      }
      // If the generic handler indicates an invalid operand
      // failure, check for a special case.
      if (Diag != Match_Success) {
        unsigned TargetDiag = validateTargetOperandClass(Actual, Formal);
        if (TargetDiag == Match_Success) {
          DEBUG_WITH_TYPE("asm-matcher",
                          dbgs() << "match success using target matcher\n");
          ++ActualIdx;
          continue;
        }
        // If the target matcher returned a specific error code use
        // that, else use the one from the generic matcher.
        if (TargetDiag != Match_InvalidOperand && HasRequiredFeatures)
          Diag = TargetDiag;
      }
      // If current formal operand wasn't matched and it is optional
      // then try to match next formal operand
      if (Diag == Match_InvalidOperand && isSubclass(Formal, OptionalMatchClass)) {
        DEBUG_WITH_TYPE("asm-matcher", dbgs() << "ignoring optional operand\n");
        continue;
      }
      // If this operand is broken for all of the instances of this
      // mnemonic, keep track of it so we can report loc info.
      // If we already had a match that only failed due to a
      // target predicate, that diagnostic is preferred.
      if (!HadMatchOtherThanPredicate &&
          (it == MnemonicRange.first || ErrorInfo <= ActualIdx)) {
        if (HasRequiredFeatures && (ErrorInfo != ActualIdx || Diag != Match_InvalidOperand))
          RetCode = Diag;
        ErrorInfo = ActualIdx;
      }
      // Otherwise, just reject this instance of the mnemonic.
      OperandsValid = false;
      break;
    }

    if (!OperandsValid) {
      DEBUG_WITH_TYPE("asm-matcher", dbgs() << "Opcode result: multiple "
                                               "operand mismatches, ignoring "
                                               "this opcode\n");
      continue;
    }
    if (!HasRequiredFeatures) {
      HadMatchOtherThanFeatures = true;
      FeatureBitset NewMissingFeatures = RequiredFeatures & ~AvailableFeatures;
      DEBUG_WITH_TYPE("asm-matcher", dbgs() << "Missing target features:";
                       for (unsigned I = 0, E = NewMissingFeatures.size(); I != E; ++I)
                         if (NewMissingFeatures[I])
                           dbgs() << ' ' << I;
                       dbgs() << "\n");
      if (NewMissingFeatures.count() <=
          MissingFeatures.count())
        MissingFeatures = NewMissingFeatures;
      continue;
    }

    Inst.clear();

    Inst.setOpcode(it->Opcode);
    // We have a potential match but have not rendered the operands.
    // Check the target predicate to handle any context sensitive
    // constraints.
    // For example, Ties that are referenced multiple times must be
    // checked here to ensure the input is the same for each match
    // constraints. If we leave it any later the ties will have been
    // canonicalized
    unsigned MatchResult;
    if ((MatchResult = checkEarlyTargetMatchPredicate(Inst, Operands)) != Match_Success) {
      Inst.clear();
      DEBUG_WITH_TYPE(
          "asm-matcher",
          dbgs() << "Early target match predicate failed with diag code "
                 << MatchResult << "\n");
      RetCode = MatchResult;
      HadMatchOtherThanPredicate = true;
      continue;
    }

    if (matchingInlineAsm) {
      convertToMapAndConstraints(it->ConvertFn, Operands);
      if (!checkAsmTiedOperandConstraints(*this, it->ConvertFn, Operands, ErrorInfo))
        return Match_InvalidTiedOperand;

      return Match_Success;
    }

    // We have selected a definite instruction, convert the parsed
    // operands into the appropriate MCInst.
    convertToMCInst(it->ConvertFn, Inst, it->Opcode, Operands);

    // We have a potential match. Check the target predicate to
    // handle any context sensitive constraints.
    if ((MatchResult = checkTargetMatchPredicate(Inst)) != Match_Success) {
      DEBUG_WITH_TYPE("asm-matcher",
                      dbgs() << "Target match predicate failed with diag code "
                             << MatchResult << "\n");
      Inst.clear();
      RetCode = MatchResult;
      HadMatchOtherThanPredicate = true;
      continue;
    }

    if (!checkAsmTiedOperandConstraints(*this, it->ConvertFn, Operands, ErrorInfo))
      return Match_InvalidTiedOperand;

    DEBUG_WITH_TYPE(
        "asm-matcher",
        dbgs() << "Opcode result: complete match, selecting this opcode\n");
    return Match_Success;
  }

  // Okay, we had no match.  Try to return a useful error code.
  if (HadMatchOtherThanPredicate || !HadMatchOtherThanFeatures)
    return RetCode;

  ErrorInfo = 0;
  return Match_MissingFeature;
}
```

接着，分析两个类MCInstrDesc和MCOperandInfo。从代码中，可以看出LQP指令，其实已经按照不同的寄存器组生成了相应的OperandInfo111[]和RISCVInsts[]。

```
static const MCOperandInfo OperandInfo111[] = { { RISCV::GPRA0RegClassID, 0, MCOI::OPERAND_REGISTER, 0 }, { RISCV::GPRNOA0A1A2A3RegClassID, 0, MCOI::OPERAND_REGISTER, 0 }, { RISCV::GPRNOA0A1A2A3RegClassID, 0, MCOI::OPERAND_REGISTER, 0 }, { -1, 0, RISCVOp::OPERAND_SIMM5, 0 }, };

{ 457,	4,	1,	4,	0,	0|(1ULL<<MCID::MayLoad), 0x12ULL, nullptr, nullptr, OperandInfo111, -1 ,nullptr },  // Inst #457 = LQP
```

///////////////我是分割线/////////////2020-04-07//////////////////

今天终于忍不住要在llvm-dev上面发问了，很快就得到了回复(Aaron Smith)，立刻准备试一下。

```
The register class for inline asm is determined by RISCVTargetLowering::getRegForInlineAsmConstraint().
Maybe you need to modify that method to return your new register class.
```

在文件/lib/Target/RISCV/RISCVISelLowing.cpp中，我找到了getRegForInlineAsmConstraint()函数的定义，现在的代码，返回的只有GPRReg、FPR32Reg和FPR64Reg三种。而目前的RISC-V inline asm支持的标号为6类，即A、I、J、K、f、r。

[6.47.3.1 Simple Constraints](https://gcc.gnu.org/onlinedocs/gcc/Simple-Constraints.html#Simple-Constraints)

[6.47.3.4 Machine Constraints](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html)

```
RISC-V:

A: An address operand (using a general-purpose register, without an offset).
I: A 12-bit signed integer immediate operand.
J: A zero integer immediate operand.
K: A 5-bit unsigned integer immediate operand.
f: A 32- or 64-bit floating-point register (requires F or D extension).
r: A 32- or 64-bit general-purpose register (depending on the platform XLEN).
```

```
std::pair<unsigned, const TargetRegisterClass *>
RISCVTargetLowering::getRegForInlineAsmConstraint(const TargetRegisterInfo *TRI,
                                                  StringRef Constraint,
                                                  MVT VT) const {
  // First, see if this is a constraint that directly corresponds to a
  // RISCV register class.
  if (Constraint.size() == 1) {
    switch (Constraint[0]) {
    case 'r':
      return std::make_pair(0U, &RISCV::GPRRegClass);
    case 'f':
      if (Subtarget.hasStdExtF() && VT == MVT::f32)
        return std::make_pair(0U, &RISCV::FPR32RegClass);
      if (Subtarget.hasStdExtD() && VT == MVT::f64)
        return std::make_pair(0U, &RISCV::FPR64RegClass);
      break;
    default:
      break;
    }
  }

  // Clang will correctly decode the usage of register name aliases into their
  // official names. However, other frontends like `rustc` do not. This allows
  // users of these frontends to use the ABI names for registers in LLVM-style
  // register constraints.
  Register XRegFromAlias = StringSwitch<Register>(Constraint.lower())
                               .Case("{zero}", RISCV::X0)
                               .Case("{ra}", RISCV::X1)
                               .Case("{sp}", RISCV::X2)
                               .Case("{gp}", RISCV::X3)
                               .Case("{tp}", RISCV::X4)
                               .Case("{t0}", RISCV::X5)
                               .Case("{t1}", RISCV::X6)
                               .Case("{t2}", RISCV::X7)
                               .Cases("{s0}", "{fp}", RISCV::X8)
                               .Case("{s1}", RISCV::X9)
                               .Case("{a0}", RISCV::X10)
                               .Case("{a1}", RISCV::X11)
                               .Case("{a2}", RISCV::X12)
                               .Case("{a3}", RISCV::X13)
                               .Case("{a4}", RISCV::X14)
                               .Case("{a5}", RISCV::X15)
                               .Case("{a6}", RISCV::X16)
                               .Case("{a7}", RISCV::X17)
                               .Case("{s2}", RISCV::X18)
                               .Case("{s3}", RISCV::X19)
                               .Case("{s4}", RISCV::X20)
                               .Case("{s5}", RISCV::X21)
                               .Case("{s6}", RISCV::X22)
                               .Case("{s7}", RISCV::X23)
                               .Case("{s8}", RISCV::X24)
                               .Case("{s9}", RISCV::X25)
                               .Case("{s10}", RISCV::X26)
                               .Case("{s11}", RISCV::X27)
                               .Case("{t3}", RISCV::X28)
                               .Case("{t4}", RISCV::X29)
                               .Case("{t5}", RISCV::X30)
                               .Case("{t6}", RISCV::X31)
                               .Default(RISCV::NoRegister);
  if (XRegFromAlias != RISCV::NoRegister)
    return std::make_pair(XRegFromAlias, &RISCV::GPRRegClass);

  // Since TargetLowering::getRegForInlineAsmConstraint uses the name of the
  // TableGen record rather than the AsmName to choose registers for InlineAsm
  // constraints, plus we want to match those names to the widest floating point
  // register type available, manually select floating point registers here.
  //
  // The second case is the ABI name of the register, so that frontends can also
  // use the ABI names in register constraint lists.
  if (Subtarget.hasStdExtF() || Subtarget.hasStdExtD()) {
    std::pair<Register, Register> FReg =
        StringSwitch<std::pair<Register, Register>>(Constraint.lower())
            .Cases("{f0}", "{ft0}", {RISCV::F0_F, RISCV::F0_D})
            .Cases("{f1}", "{ft1}", {RISCV::F1_F, RISCV::F1_D})
            .Cases("{f2}", "{ft2}", {RISCV::F2_F, RISCV::F2_D})
            .Cases("{f3}", "{ft3}", {RISCV::F3_F, RISCV::F3_D})
            .Cases("{f4}", "{ft4}", {RISCV::F4_F, RISCV::F4_D})
            .Cases("{f5}", "{ft5}", {RISCV::F5_F, RISCV::F5_D})
            .Cases("{f6}", "{ft6}", {RISCV::F6_F, RISCV::F6_D})
            .Cases("{f7}", "{ft7}", {RISCV::F7_F, RISCV::F7_D})
            .Cases("{f8}", "{fs0}", {RISCV::F8_F, RISCV::F8_D})
            .Cases("{f9}", "{fs1}", {RISCV::F9_F, RISCV::F9_D})
            .Cases("{f10}", "{fa0}", {RISCV::F10_F, RISCV::F10_D})
            .Cases("{f11}", "{fa1}", {RISCV::F11_F, RISCV::F11_D})
            .Cases("{f12}", "{fa2}", {RISCV::F12_F, RISCV::F12_D})
            .Cases("{f13}", "{fa3}", {RISCV::F13_F, RISCV::F13_D})
            .Cases("{f14}", "{fa4}", {RISCV::F14_F, RISCV::F14_D})
            .Cases("{f15}", "{fa5}", {RISCV::F15_F, RISCV::F15_D})
            .Cases("{f16}", "{fa6}", {RISCV::F16_F, RISCV::F16_D})
            .Cases("{f17}", "{fa7}", {RISCV::F17_F, RISCV::F17_D})
            .Cases("{f18}", "{fs2}", {RISCV::F18_F, RISCV::F18_D})
            .Cases("{f19}", "{fs3}", {RISCV::F19_F, RISCV::F19_D})
            .Cases("{f20}", "{fs4}", {RISCV::F20_F, RISCV::F20_D})
            .Cases("{f21}", "{fs5}", {RISCV::F21_F, RISCV::F21_D})
            .Cases("{f22}", "{fs6}", {RISCV::F22_F, RISCV::F22_D})
            .Cases("{f23}", "{fs7}", {RISCV::F23_F, RISCV::F23_D})
            .Cases("{f24}", "{fs8}", {RISCV::F24_F, RISCV::F24_D})
            .Cases("{f25}", "{fs9}", {RISCV::F25_F, RISCV::F25_D})
            .Cases("{f26}", "{fs10}", {RISCV::F26_F, RISCV::F26_D})
            .Cases("{f27}", "{fs11}", {RISCV::F27_F, RISCV::F27_D})
            .Cases("{f28}", "{ft8}", {RISCV::F28_F, RISCV::F28_D})
            .Cases("{f29}", "{ft9}", {RISCV::F29_F, RISCV::F29_D})
            .Cases("{f30}", "{ft10}", {RISCV::F30_F, RISCV::F30_D})
            .Cases("{f31}", "{ft11}", {RISCV::F31_F, RISCV::F31_D})
            .Default({RISCV::NoRegister, RISCV::NoRegister});
    if (FReg.first != RISCV::NoRegister)
      return Subtarget.hasStdExtD()
                 ? std::make_pair(FReg.second, &RISCV::FPR64RegClass)
                 : std::make_pair(FReg.first, &RISCV::FPR32RegClass);
  }

  return TargetLowering::getRegForInlineAsmConstraint(TRI, Constraint, VT);
}
```

为了满足Haawking DSC的需求，现在新增三个算下，h代表GPRA0，x代表GPRNOA0A1，g代表GPRNOA0A1A2A3，修改代码如下：

```
if (Constraint.size() == 1) {
    switch (Constraint[0]) {
    case 'r':
      return std::make_pair(0U, &RISCV::GPRRegClass);
    case 'h':
      return std::make_pair(0U, &RISCV::GPRA0RegClass);
    case 'x':
      return std::make_pair(0U, &RISCV::GPRNOA0A1RegClass);
    case 'g':
      return std::make_pair(0U, &RISCV::GPRNOA0A1A2A3RegClass);
    case 'f':
      if (Subtarget.hasStdExtF() && VT == MVT::f32)
        return std::make_pair(0U, &RISCV::FPR32RegClass);
      if (Subtarget.hasStdExtD() && VT == MVT::f64)
        return std::make_pair(0U, &RISCV::FPR64RegClass);
      break;
    default:
      break;
    }
  }
```

修改C程序如下，在编译生成LLVM IR结果的时候，报出错误，不识别h等字符，因此需要进行相应的修改：

```
  asm volatile
  (
    "lqp   %[z], %[x], %[y], 4\n\t"
    : [z] "=h" (c)
    : [x] "g" (a), [y] "g" (b)
  ) ;


error: invalid output constraint '=h' in asm
```

///////////////我是分割线/////////////2020-04-16//////////////////

今天终于调试出来了，目前看来结果与预期的相一致。首先贴上来编译后的三条指令，LQP，LDP，SDP。

```
007e0124 main:
  7e0124: 41 11                         addi sp, sp, -16
  7e0126: 06 c6                         sw ra, 12(sp)
  7e0128: 15 47                         addi a4, zero, 5
  7e012a: 2b 35 c7 ff                   ldp a0, -4(a4)
  7e012e: 2a c4                         sw a0, 8(sp)
  7e0130: 2b 7e a7 fe                   sdp a0, -4(a4)
  7e0134: 2a c4                         sw a0, 8(sp)
  7e0136: 89 47                         addi a5, zero, 2
  7e0138: 0b 05 f7 c8                   lqp a0, a4, a5, 4
  7e013c: 2a c4                         sw a0, 8(sp)
  7e013e: 49 65                         lui a0, 18
  7e0140: 71 15                         addi a0, a0, -4
  7e0142: b7 c5 ad de                   lui a1, 912092
  7e0146: 93 85 f5 ee                   addi a1, a1, -273
  7e014a: ef 10 b0 08                   jal 6282
  7e014e: 01 45                         mv a0, zero
  7e0150: b2 40                         lw ra, 12(sp)
  7e0152: 41 01                         addi sp, sp, 16
  7e0154: 82 80                         ret
```

需要修改的文件只有一个/lib/Target/RISCV/RISCVISelLowing.cpp，（得到这个结论，废了老鼻子劲了）。

```
RISCVTargetLowering::ConstraintType
RISCVTargetLowering::getConstraintType(StringRef Constraint) const {
  if (Constraint.size() == 1) {
    switch (Constraint[0]) {
    default:
      break;
    case 'd':
      return C_RegisterClass;
    case 'e':
      return C_RegisterClass;
    case 'f':
      return C_RegisterClass;
    case 'I':
    case 'J':
    case 'K':
      return C_Immediate;
    case 'A':
      return C_Memory;
    }
  }
  return TargetLowering::getConstraintType(Constraint);
}
```

```
// First, see if this is a constraint that directly corresponds to a
  // RISCV register class.
  if (Constraint.size() == 1) {
    switch (Constraint[0]) {
    case 'r':
      return std::make_pair(0U, &RISCV::GPRRegClass);
    case 'X':
      return std::make_pair(0U, &RISCV::GPRA0RegClass);
    case 'd':
      return std::make_pair(0U, &RISCV::GPRNoA0A1RegClass);
    case 'e':
      return std::make_pair(0U, &RISCV::GPRNoA0A1A2A3RegClass);
    case 'f':
      if (Subtarget.hasStdExtF() && VT == MVT::f32)
        return std::make_pair(0U, &RISCV::FPR32RegClass);
      if (Subtarget.hasStdExtD() && VT == MVT::f64)
        return std::make_pair(0U, &RISCV::FPR64RegClass);
      break;
    default:
      break;
    }
  }
```

至此，这个工作也到一段落了。最近一周搞得头都大了，用的命令也就是一个find，有问题就查找错误信息，然后进行修改。甚至还修改了一些clang的文件，但是后来发现，这个方法是可行的。下一步就是继续研究，目前的效果只是能够选择A0而不让其他选择A0A1，A0A1A2A3等。
（Junning Wu, At Beijing Xigema B106，20200416）