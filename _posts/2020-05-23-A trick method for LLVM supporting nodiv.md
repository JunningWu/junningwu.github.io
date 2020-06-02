---
layout: post
title: A trick method for LLVM supporting nodiv
date: 2020-05-23
categories: Tech
tags: [Essay,Tech]
description: 由于大多数嵌入式MCU和DSP受限于成本预算，均未内置硬件除法器，因此GCC有一个编译选项-nodiv，但是RISC-V的LLVM后端，目前还未实现这个选项，现在使用了一个取巧的方式，既支持乘法指令，又不支持除法操作。
---

由于大多数嵌入式MCU和DSP受限于成本预算，均未内置硬件除法器，因此GCC有一个编译选项-nodiv，但是RISC-V的LLVM后端，目前还未实现这个选项，现在使用了一个取巧的方式，既支持乘法指令，又不支持除法操作。

而对于除法操作的支持，还有浮点乘除法的支持，通过调用标准算法库实现。当然，有能力的企业，如ARM和TI，则会自己编写运行时库来支持不同格式的除法操作。

在RISC-V当前版本的LLVM后端代码中，是通过HasStdExtM标志实现标准扩展指令集M的，具体文件为llvm/lib/Target/RISCV/RISCVInstrInfoM.td。

```
let Predicates = [HasStdExtM] in {
def MUL     : ALU_rr<0b0000001, 0b000, "mul">,
              Sched<[WriteIMul, ReadIMul, ReadIMul]>;
def MULH    : ALU_rr<0b0000001, 0b001, "mulh">,
              Sched<[WriteIMul, ReadIMul, ReadIMul]>;
def MULHSU  : ALU_rr<0b0000001, 0b010, "mulhsu">,
              Sched<[WriteIMul, ReadIMul, ReadIMul]>;
def MULHU   : ALU_rr<0b0000001, 0b011, "mulhu">,
              Sched<[WriteIMul, ReadIMul, ReadIMul]>;
def DIV     : ALU_rr<0b0000001, 0b100, "div">,
              Sched<[WriteIDiv, ReadIDiv, ReadIDiv]>;
def DIVU    : ALU_rr<0b0000001, 0b101, "divu">,
              Sched<[WriteIDiv, ReadIDiv, ReadIDiv]>;
def REM     : ALU_rr<0b0000001, 0b110, "rem">,
              Sched<[WriteIDiv, ReadIDiv, ReadIDiv]>;
def REMU    : ALU_rr<0b0000001, 0b111, "remu">,
              Sched<[WriteIDiv, ReadIDiv, ReadIDiv]>;
} // Predicates = [HasStdExtM]
```

完全可以通过增加一个类似于GCC的编译选项nodiv实现对没有硬件除法器的支持，但是受限于个人能力和精力，选取了这种取巧的方式。将乘法指令相关的定义提出来，不依赖于HasStdExtM标志。在编译应用程序的时候，不需要添加M标志，就可以生成乘法指令。

```
def MUL     : ALU_rr<0b0000001, 0b000, "mul">,
              Sched<[WriteIMul, ReadIMul, ReadIMul]>;
def MULH    : ALU_rr<0b0000001, 0b001, "mulh">,
              Sched<[WriteIMul, ReadIMul, ReadIMul]>;
def MULHSU  : ALU_rr<0b0000001, 0b010, "mulhsu">,
              Sched<[WriteIMul, ReadIMul, ReadIMul]>;
def MULHU   : ALU_rr<0b0000001, 0b011, "mulhu">,
              Sched<[WriteIMul, ReadIMul, ReadIMul]>;
```

当然，为了完成应用程序的正常执行，还需要在编译Libc库和newlib库的时候，不能产生除法指令，否则依然无法执行。

这一点可以通过RV32i进行交叉编译，也可以通过新生成的编译器进行交叉编译。同样的，为了方便，我这里选择了前一种。

希望后面有精力的时候，可以为LLVM社区贡献nodiv的支持。

（2020-05-23，西格玛公寓，北京）