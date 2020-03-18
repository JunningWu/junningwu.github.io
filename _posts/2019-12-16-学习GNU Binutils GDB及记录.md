---
layout: post
title: 学习GNU Binutils GDB及记录
date: 2019-12-16
categories: Tech
tags: [Eaasy,Tech]
description: RISC-V GNU Binutils GDB新增自定义指令相关源码分析。
---
```
目录
## RISC-V opcode 结构体
## RISC-V Opcode Argumets含义
## Gnu Binutils as RISC-V Dependent Features
### RISC-V指令格式中用到的OPCODE名称
### RISC-V指令格式中用到的标号
### RISC-V指令格式
### RISC-V相关的选项
## Pulp RISCY新增指令
### Post-Incrementing Load & Store Instructions
### Hardware Loops
### ALU
### Multiply-Accumulate
### Vectorial
## GNU Binutils Commands with Examples
### as - GNU Assembler Command
### ld – GNU Linker Command
### ar – GNU Archive Command
### nm – List Object File Symbols
### objcopy – Copy and Translate Object Files
### objdump – Display Object File Information
### size – List Section Size and Toal Size
### strings – Display Printable Characters from a File
### strip – Discard Symbols from Object File
### c++filt – Demangle Command
### addr2line – Convert Address to Filename and Numbers
### readelf – Display ELF File Info
```

基于RISC-V基本指令集增加自定义指令的过程，可以查看我的前一篇博文《使用Gem5自定义RISC-V指令集（持续更新）》。这篇博文主要是记录一些相关源码的分析、理解和注释。

## RISC-V opcode 结构体的定义
```
/* This structure holds information for a particular instruction.  */

struct riscv_opcode
{
  /* The name of the instruction.  */
  const char *name;
  /* The ISA subset name (I, M, A, F, D, Xextension).  */
  const char *subset;
  /* A string describing the arguments for this instruction.  */
  const char *args;
  /* The basic opcode for the instruction.  When assembling, this
     opcode is modified by the arguments to produce the actual opcode
     that is used.  If pinfo is INSN_MACRO, then this is 0.  */
  insn_t match;
  /* If pinfo is not INSN_MACRO, then this is a bit mask for the
     relevant portions of the opcode when disassembling.  If the
     actual opcode anded with the match field equals the opcode field,
     then we have found the correct instruction.  If pinfo is
     INSN_MACRO, then this field is the macro identifier.  */
  insn_t mask;
  /* A function to determine if a word corresponds to this instruction.
     Usually, this computes ((word & mask) == match).  */
  int (*match_func) (const struct riscv_opcode *op, insn_t word);
  /* For a macro, this is INSN_MACRO.  Otherwise, it is a collection
     of bits describing the instruction, notably any relevant hazard
     information.  */
  unsigned long pinfo;
};
```

## RISC-V Opcode Argumets含义

```
   The order of overloaded instructions matters.  Label arguments and
   register arguments look the same. Instructions that can have either
   for arguments must apear in the correct order in this table for the
   assembler to pick the right one. In other words, entries with
   immediate operands must apear after the same instruction with
   registers.

   Because of the lookup algorithm used, entries with the same opcode
   name must be contiguous.  
```

- C
RVC的意思
- s/w
RVC，RS1 x8-x15
- t/x
RVC，RS2 x8-x15
- U
RVC，RS1，与RD相同
- c
RVC，RS1，与sp相同
- V
RVC，RS2
- i
RVC，RVC_SIMM3
- o/j
RVC，RVC_IMM
- k
RVC，RVC_LW_IMM
- l
RVC，RVC_LD_IMM
- m
RVC，RVC_LWSP_IMM
- n
RVC，RVC_LDSP_IMM
- K
RVC_ADDI4SPN_IMM
- L
RVC_ADDI16SP_IMM
- M
RVC_SWSP_IMM
- N
RVC_SDSP_IMM
- p
RVC，info->target = EXTRACT_RVC_B_IMM (l) + pc;
(*info->print_address_func) (info->target, info);
- a
RVC，info->target = EXTRACT_RVC_J_IMM (l) + pc;
(*info->print_address_func) (info->target, info);
- u
RVC，EXTRACT_RVC_IMM (l) & (RISCV_BIGIMM_REACH-1)
- >
RVC，EXTRACT_RVC_IMM (l) & 0x3f
- <
RVC，EXTRACT_RVC_IMM (l) & 0x1f
- T
RVC，floating-point RS2, EXTRACT_OPERAND (CRS2, l)
- D
RVC，floating-point RS2 x8-x15, EXTRACT_OPERAND (CRS2S, l) + 8

- s
riscv_gpr_names[rs1]
- t
EXTRACT_OPERAND (RS2, l)
- u
EXTRACT_UTYPE_IMM (l) >> RISCV_IMM_BITS
- m
arg_print (info, EXTRACT_OPERAND (RM, l), riscv_rm, ARRAY_SIZE (riscv_rm));
- P
arg_print (info, EXTRACT_OPERAND (PRED, l), riscv_pred_succ, ARRAY_SIZE (riscv_pred_succ));
- Q
arg_print (info, EXTRACT_OPERAND (SUCC, l), riscv_pred_succ, ARRAY_SIZE (riscv_pred_succ));
- o
EXTRACT_ITYPE_IMM (l)
- j
if (((l & MASK_ADDI) == MATCH_ADDI && rs1 != 0) || (l & MASK_JALR) == MATCH_JALR) maybe_print_address (pd, rs1, EXTRACT_ITYPE_IMM (l));
- q
maybe_print_address (pd, rs1, EXTRACT_STYPE_IMM (l));
- a
info->target = EXTRACT_UJTYPE_IMM (l) + pc;
- p
info->target = EXTRACT_SBTYPE_IMM (l) + pc;
- d
```
if ((l & MASK_AUIPC) == MATCH_AUIPC)
  pd->hi_addr[rd] = pc + EXTRACT_UTYPE_IMM (l);
else if ((l & MASK_LUI) == MATCH_LUI)
  pd->hi_addr[rd] = EXTRACT_UTYPE_IMM (l);
else if ((l & MASK_C_LUI) == MATCH_C_LUI)
  pd->hi_addr[rd] = EXTRACT_RVC_LUI_IMM (l);
```
- z
riscv_gpr_names[0]
- >
EXTRACT_OPERAND (SHAMT, l)
- <
EXTRACT_OPERAND (SHAMTW, l)
- S/U
riscv_fpr_names[rs1]
- T
riscv_fpr_names[EXTRACT_OPERAND (RS2, l)]
- D
riscv_fpr_names[rd]
- R
riscv_fpr_names[EXTRACT_OPERAND (RS3, l)]
- E
```
const char* csr_name = NULL;
unsigned int csr = EXTRACT_OPERAND (CSR, l);
switch (csr)
  {
    #define DECLARE_CSR(name, num) case num: csr_name = #name; break;
    #include "opcode/riscv-opc.h"
    #undef DECLARE_CSR
  }
if (csr_name)
  print (info->stream, "%s", csr_name);
else
  print (info->stream, "0x%x", csr);
break;
```
在gas/config/tc-riscv.c文件中，也有对这些oprand的解释，具体如下(这里参考的是ri5cy_gnu_toolchain-master中的代码)：
```
/* For consistency checking, verify that all bits are specified either
   by the match/mask part of the instruction definition, or by the
   operand list.  */
static int
validate_riscv_insn (const struct riscv_opcode *opc)
{
  const char *p = opc->args;
  char c;
  insn_t used_bits = opc->mask;
  int insn_width = 8 * riscv_insn_length (opc->match);
  insn_t required_bits = ~0ULL >> (64 - insn_width);

  if ((used_bits & opc->match) != (opc->match & required_bits))
    {
      as_bad (_("internal: bad RISC-V opcode (mask error): %s %s. Used bits: %lX, Match bits: %lX, Required bits: %lX, Eval: %lX, Insn width=%d"),
	      opc->name, opc->args, used_bits, opc->match, required_bits, (used_bits & opc->match), insn_width);
      return 0;
    }

#define USE_BITS(mask,shift)	(used_bits |= ((insn_t)(mask) << (shift)))
  while (*p)
    switch (c = *p++)
      {
      /* Xcustom */
      case '^':
	switch (c = *p++)
	  {
	  case 'd': USE_BITS (OP_MASK_RD, OP_SH_RD); break;
	  case 's': USE_BITS (OP_MASK_RS1, OP_SH_RS1); break;
	  case 't': USE_BITS (OP_MASK_RS2, OP_SH_RS2); break;
	  case 'j': USE_BITS (OP_MASK_CUSTOM_IMM, OP_SH_CUSTOM_IMM); break;
	  }
	break;
      case 'C': /* RVC */
	switch (c = *p++)
	  {
	  case 'a': used_bits |= ENCODE_RVC_J_IMM(-1U); break;
	  case 'c': break; /* RS1, constrained to equal sp */
	  case 'i': used_bits |= ENCODE_RVC_SIMM3(-1U); break;
	  case 'j': used_bits |= ENCODE_RVC_IMM(-1U); break;
	  case 'k': used_bits |= ENCODE_RVC_LW_IMM(-1U); break;
	  case 'l': used_bits |= ENCODE_RVC_LD_IMM(-1U); break;
	  case 'm': used_bits |= ENCODE_RVC_LWSP_IMM(-1U); break;
	  case 'n': used_bits |= ENCODE_RVC_LDSP_IMM(-1U); break;
	  case 'p': used_bits |= ENCODE_RVC_B_IMM(-1U); break;
	  case 's': USE_BITS (OP_MASK_CRS1S, OP_SH_CRS1S); break;
	  case 't': USE_BITS (OP_MASK_CRS2S, OP_SH_CRS2S); break;
	  case 'u': used_bits |= ENCODE_RVC_IMM(-1U); break;
	  case 'v': used_bits |= ENCODE_RVC_IMM(-1U); break;
	  case 'w': break; /* RS1S, constrained to equal RD */
	  case 'x': break; /* RS2S, constrained to equal RD */
	  case 'K': used_bits |= ENCODE_RVC_ADDI4SPN_IMM(-1U); break;
	  case 'L': used_bits |= ENCODE_RVC_ADDI16SP_IMM(-1U); break;
	  case 'M': used_bits |= ENCODE_RVC_SWSP_IMM(-1U); break;
	  case 'N': used_bits |= ENCODE_RVC_SDSP_IMM(-1U); break;
	  case 'U': break; /* RS1, constrained to equal RD */
	  case 'V': USE_BITS (OP_MASK_CRS2, OP_SH_CRS2); break;
	  case '<': used_bits |= ENCODE_RVC_IMM(-1U); break;
	  case '>': used_bits |= ENCODE_RVC_IMM(-1U); break;
	  case 'T': USE_BITS (OP_MASK_CRS2, OP_SH_CRS2); break;
	  case 'D': USE_BITS (OP_MASK_CRS2S, OP_SH_CRS2S); break;
	  default:
	    as_bad (_("internal: bad RISC-V opcode (unknown operand type `C%c'): %s %s"),
		    c, opc->name, opc->args);
	    return 0;
	  }
	break;
      case ',': break;
      case '(': break;
      case ')': break;
      case '!': break;
      case '<': USE_BITS (OP_MASK_SHAMTW,	OP_SH_SHAMTW);	break;
      case '>':	USE_BITS (OP_MASK_SHAMT,	OP_SH_SHAMT);	break;
      case 'A': break;
      case 'D':	USE_BITS (OP_MASK_RD,		OP_SH_RD);	break;
      case 'Z':	USE_BITS (OP_MASK_RS1,		OP_SH_RS1);	break;
      case 'E':	USE_BITS (OP_MASK_CSR,		OP_SH_CSR);	break;
      case 'I': break;
      case 'R':	USE_BITS (OP_MASK_RS3,		OP_SH_RS3);	break;
      case 'S':	USE_BITS (OP_MASK_RS1,		OP_SH_RS1);	break;
      case 'U':	USE_BITS (OP_MASK_RS1,		OP_SH_RS1);	/* fallthru */
      case 'T':	USE_BITS (OP_MASK_RS2,		OP_SH_RS2);	break;
      case 'd':	USE_BITS (OP_MASK_RD,		OP_SH_RD);      if (*p == 'i') ++p;  break;
      case 'm':	USE_BITS (OP_MASK_RM,		OP_SH_RM);	break;
      case 's':	USE_BITS (OP_MASK_RS1,		OP_SH_RS1);	break;
      case 't':	USE_BITS (OP_MASK_RS2,		OP_SH_RS2);	break;
      case 'r': USE_BITS (OP_MASK_RS3I,         OP_SH_RS3I);    break;
      case 'P':	USE_BITS (OP_MASK_PRED,		OP_SH_PRED); break;
      case 'Q':	USE_BITS (OP_MASK_SUCC,		OP_SH_SUCC); break;
      case 'o':
      case 'j': used_bits |= ENCODE_ITYPE_IMM(-1U); if (*p == 'i') ++p; break;
      case 'a':	used_bits |= ENCODE_UJTYPE_IMM(-1U); break;
      case 'p':	used_bits |= ENCODE_SBTYPE_IMM(-1U); break;
      case 'q':	used_bits |= ENCODE_STYPE_IMM(-1U); break;
      case 'u':	used_bits |= ENCODE_UTYPE_IMM(-1U); break;
      case '[': break;
      case ']': break;
      case '0': break;
      case 'b': 
		if (*p == '1') {
			used_bits |= ENCODE_ITYPE_IMM(-1U); /* For loop I type pc rel displacement */
			++p; break;
		} else if (*p == '2') {
			used_bits |= ENCODE_I1TYPE_UIMM(-1U); /* For loop I1 type pc rel displacement */
			++p; break;
		} else if (*p == '3') {
			used_bits |= ENCODE_I1TYPE_UIMM(-1U); /* For scallimm  */
			++p; break;
		} else if (*p == '5') {
			used_bits |= ENCODE_I5TYPE_UIMM(-1U);
			++p; break;
		} else if (*p == 'i') {
			used_bits |= ENCODE_I5_1_TYPE_UIMM(-1U);
			++p; break;
		} else if (*p == 'I') {
			used_bits |= ENCODE_I5_1_TYPE_IMM(-1U);
			++p; break;
		} else if (*p == 's' || *p == 'u' || *p == 'U' || *p == 'f' || *p == 'F') {
			used_bits |= ENCODE_I6TYPE_IMM(-1U);
			++p; break;
		}
      default:
	as_bad (_("internal: bad RISC-V opcode (unknown operand type `%c'): %s %s"),
		c, opc->name, opc->args);
	return 0;
      }
#undef USE_BITS
  if (used_bits != required_bits)
    {
      as_bad (_("internal: bad RISC-V opcode (bits 0x%lx undefined): %s %s, width=%d"),
	      ~(long)(used_bits & required_bits), opc->name, opc->args, insn_width);
      return 0;
    }
  return 1;
}
```

## Gnu Binutils as RISC-V Dependent Features
[9.38 RISC-V Dependent Features](https://sourceware.org/binutils/docs/as/RISC_002dV_002dDependent.html#RISC_002dV_002dDependent)
### RISC-V指令格式中用到的标号

- opcode Unsigned immediate or opcode name for 7-bits opcode.
- opcode2 Unsigned immediate or opcode name for 2-bits opcode. 主要用于C指令集
- func7 Unsigned immediate for 7-bits function code. I指令集的R类型
- func6 Unsigned immediate for 6-bits function code. 主要用于C指令集CA类型
- func4 Unsigned immediate for 4-bits function code. 主要用于C指令集CR类型
- func3 Unsigned immediate for 3-bits function code. 
- func2 Unsigned immediate for 2-bits function code. R4类型和C指令集CR类型
- rd Destination register number for operand x, can be GPR or FPR.
- rd’ Destination register number for operand x,only accept s0-s1, a0-a5, fs0-fs1 and fa0-fa5. 主要用于C指令集CIW、CA类型
- rs1 First source register number for operand x, can be GPR or FPR.
- rs1’ First source register number for operand x,only accept s0-s1, a0-a5, fs0-fs1 and fa0-fa5. 主要用于C指令集CB、CA类型
- rs2 Second source register number for operand x, can be GPR or FPR.
- rs2’ Second source register number for operand x,only accept s0-s1, a0-a5, fs0-fs1 and fa0-fa5.主要用于C指令集CA类型
- simm12 Sign-extended 12-bit immediate for operand x.
- simm20 Sign-extended 20-bit immediate for operand x.
- simm6 Sign-extended 6-bit immediate for operand x.主要用于C指令集CI类型
- uimm8 Unsigned 8-bit immediate for operand x.主要用于C指令集CIW类型
- symbol Symbol or lable reference for operand x.

### RISC-V指令格式中用到的OPCODE名称
- C0
- C1
- C2
Opcode space for compressed instructions.
- LOAD
Opcode space for load instructions.
- LOAD_FP
Opcode space for floating-point load instructions.
- STORE
Opcode space for store instructions.
- STORE_FP
Opcode space for floating-point store instructions.
- AUIPC
Opcode space for auipc instruction.
- LUI
Opcode space for lui instruction.
- BRANCH
Opcode space for branch instructions.
- JAL
Opcode space for jal instruction.
- JALR
Opcode space for jalr instruction.
- OP
Opcode space for ALU instructions.
- OP_32
Opcode space for 32-bits ALU instructions.
- OP_IMM
Opcode space for ALU with immediate instructions.
- OP_IMM_32
Opcode space for 32-bits ALU with immediate instructions.
- OP_FP
Opcode space for floating-point operation instructions.
- MADD
Opcode space for madd instruction.
- MSUB
Opcode space for msub instruction.
- NMADD
Opcode space for nmadd instruction.
- NMSUB
Opcode space for msub instruction.
- AMO
Opcode space for atomic memory operation instructions.
- MISC_MEM
Opcode space for misc instructions.
- SYSTEM
Opcode space for system instructions.
- CUSTOM_0
- CUSTOM_1
- CUSTOM_2
- CUSTOM_3
Opcode space for customize instructions.

### RISC-V指令格式
RISC-V的指令为2字节或者4字节，而且需要2字节对齐；指令编码的低两位指示出指令长度，00/01/10表示2字节，11表示4字节。

R type: .insn r opcode, func3, func7, rd, rs1, rs2

+-------+-----+-----+-------+----+-------------+
| func7 | rs2 | rs1 | func3 | rd |      opcode |
+-------+-----+-----+-------+----+-------------+
31      25    20    15      12   7             0

R type with 4 register operands: .insn r opcode, func3, func2, rd, rs1, rs2, rs3

R4 type: .insn r4 opcode, func3, func2, rd, rs1, rs2, rs3

+-----+-------+-----+-----+-------+----+-------------+
| rs3 | func2 | rs2 | rs1 | func3 | rd |      opcode |
+-----+-------+-----+-----+-------+----+-------------+
31    27      25    20    15      12   7             0


I type: .insn i opcode, func3, rd, rs1, simm12

+-------------+-----+-------+----+-------------+
|      simm12 | rs1 | func3 | rd |      opcode |
+-------------+-----+-------+----+-------------+
31            20    15      12   7             0

S type: .insn s opcode, func3, rd, rs1, simm12

+--------------+-----+-----+-------+-------------+-------------+
| simm12[11:5] | rs2 | rs1 | func3 | simm12[4:0] |      opcode |
+--------------+-----+-----+-------+-------------+-------------+
31             25    20    15      12            7             0

SB type: .insn sb opcode, func3, rd, rs1, symbol

SB type: .insn sb opcode, func3, rd, simm12(rs1)

B type: .insn s opcode, func3, rd, rs1, symbol

B type: .insn s opcode, func3, rd, simm12(rs1)

+------------+--------------+-----+-----+-------+-------------+-------------+--------+
| simm12[12] | simm12[10:5] | rs2 | rs1 | func3 | simm12[4:1] | simm12[11]] | opcode |
+------------+--------------+-----+-----+-------+-------------+-------------+--------+
31          30            25    20    15      12           7            0

U type: .insn u opcode, rd, simm20

+---------------------------+----+-------------+
|                    simm20 | rd |      opcode |
+---------------------------+----+-------------+
31                          12   7             0

UJ type: .insn uj opcode, rd, symbol

J type: .insn j opcode, rd, symbol

+------------+--------------+------------+---------------+----+-------------+
| simm20[20] | simm20[10:1] | simm20[11] | simm20[19:12] | rd |      opcode |
+------------+--------------+------------+---------------+----+-------------+
31           30             21           20              12   7             0

CR type: .insn cr opcode2, func4, rd, rs2

+---------+--------+-----+---------+
|   func4 | rd/rs1 | rs2 | opcode2 |
+---------+--------+-----+---------+
15        12       7     2        0

CI type: .insn ci opcode2, func3, rd, simm6

+---------+-----+--------+-----+---------+
|   func3 | imm | rd/rs1 | imm | opcode2 |
+---------+-----+--------+-----+---------+
15        13    12       7     2         0

CIW type: .insn ciw opcode2, func3, rd, uimm8

+---------+--------------+-----+---------+
|   func3 |          imm | rd' | opcode2 |
+---------+--------------+-----+---------+
15        13             7     2         0

CA type: .insn ca opcode2, func6, func2, rd, rs2

+---------+----------+-------+------+--------+
|   func6 | rd'/rs1' | func2 | rs2' | opcode |
+---------+----------+-------+------+--------+
15        10         7       5      2        0

CB type: .insn cb opcode2, func3, rs1, symbol

+---------+--------+------+--------+---------+
|   func3 | offset | rs1' | offset | opcode2 |
+---------+--------+------+--------+---------+
15        13       10     7        2         0

CJ type: .insn cj opcode2, symbol

+---------+--------------------+---------+
|   func3 |        jump target | opcode2 |
+---------+--------------------+---------+
15        13             7     2         0

### RISC-V相关的选项
- fpic
- fPIC
Generate position-independent code
- fno-pic
Don’t generate position-independent code (default)
- march=ISA
Select the base isa, as specified by ISA. For example -march=rv32ima.
- mabi=ABI
Selects the ABI, which is either "ilp32" or "lp64", optionally followed by "f", "d", or "q" to indicate single-precision, double-precision, or quad-precision floating-point calling convention, or none to indicate the soft-float calling convention. Also, "ilp32" can optionally be followed by "e" to indicate the RVE ABI, which is always soft-float.
- mrelax
Take advantage of linker relaxations to reduce the number of instructions required to materialize symbol addresses. (default)
- mno-relax
Don’t do linker relaxations.

## Pulp RISCY新增指令
Pulp基于RISC-V基本指令集增加五大类的扩展指令，包括Load、Store相关的指令，硬件循环HWLOOP，ALU的一些自定义指令，乘累加等DSP指令以及向量指令Vector等。

### Post-Incrementing Load & Store Instructions（24条）
Post-Incrementing load and store instructions perform a load, or a store, respectively, while at the same time
incrementing the address that was used for the memory access. Since it is a post-incrementing scheme, the
base address is used for the access and the modified address is written back to the register-file. There are
versions of those instructions that use immediates and those that use registers as offsets. The base address
always comes from a register.
### Hardware Loops（6条）
RI5CY supports 2 levels of nested hardware loops. The loop has to be setup before entering the loop body.
For this purpose, there are two methods, either the long commands that separately set start- and end-
addresses of the loop and the number of iterations, or the short command that does all of this in a single
instruction. The short command has a limited range for the number of instructions contained in the loop and
the loop must start in the next instruction after the setup instruction.
Loop number 0 has higher priority than loop number 1 in a nested loop configuration, meaning that loop 0
represents the inner loop.
A hardware loop is subject to the following constraints:
- Minimum of 2 instructions in the loop body.
- Loop counter has to be bigger than 0, since the loop body is always entered at least once.
### ALU（48条）
The ALU extensions are split into several subgroups that belong together.
- Bit manipulation instructions are useful to work on single bits or groups of bits within a word, see
Section 14.3.1.
- General ALU instructions try to fuse common used sequences into a single instruction and thus
increase the performance of small kernels that use those sequence, see Section 14.3.3.
- Immediate branching instructions are useful to compare a register with an immediate value before
taking or not a branch, see Section 13.3.5.
### Multiply-Accumulate（22条）
- 32-Bit x 32-Bit Multiplication Operations
- 16-Bit x 16-Bit Multiplication
### Vectorial（200条）
Vectorial instructions perform operations in a SIMD-like manner on multiple sub-word elements at the same
time. This is done by segmenting the data path into smaller parts when 8 or 16-bit operations should be
performed.
Vectorial instructions are available in two flavors:
- 8-Bit, to perform four operations on the 4 bytes inside a 32-bit word at the same time
- 16-Bit, to perform two operations on the 2 half-words inside a 32-bit word at the same time
Additionally, there are three modes that influence the second operand:
- 1. Normal mode, vector-vector operation. Both operands, from rs1 and rs2, are treated as vectors of
bytes or half-words.
- 2. Scalar replication mode (.sc), vector-scalar operation. Operand 1 is treated as a vector, while
operand 2 is treated as a scalar and replicated two or four times to form a complete vector. The LSP
is used for this purpose.
- 3. Immediate scalar replication mode (.sci), vector-scalar operation. Operand 1 is treated as vector,
while operand 2 is treated as a scalar and comes from an immediate. The immediate is either sign-
or zero-extended, depending on the operation. If not specified, the immediate is sign-extended.

## GNU Binutils Commands with Examples

Gnu Binutils是GNU Toolchain中重要的工具，包括下列子程序，完成汇编、链接等功能。
- as – GNU Assembler Command
- ld – GNU Linker Command
- ar – GNU Archive Command
- nm – List Object File Symbols
- objcopy – Copy and Translate Object Files
- objdump – Display Object File Information
- size – List Section Size and Toal Size
- strings – Display Printable Characters from a File
- strip – Discard Symbols from Object File
- c++filt – Demangle Command
- addr2line – Convert Address to Filename and Numbers
- readelf – Display ELF File Info

用来做测试的程序如下，比较简单，就是模拟简单的函数调用，返回1。
```
// func1.c file:
int func1() {
	return func2();
}

// func2.c file:
int func2() {
	return 1;
}

// main.c file:
int main() {
	return func1();
}
```

在进行下一步之前，我们先调用gcc -S将c程序代码转换成s文件。

- func2.s	

```
.file	"func2.c"
	.option nopic
	.text
	.align	1
	.globl	func2
	.type	func2, @function
func2:
	addi	sp,sp,-16
	sd	s0,8(sp)
	addi	s0,sp,16
	li	a5,1
	mv	a0,a5
	ld	s0,8(sp)
	addi	sp,sp,16
	jr	ra
	.size	func2, .-func2
	.ident	"GCC: (GNU) 7.2.0"
```

- func1.s

```
	.file	"func1.c"
	.option nopic
	.text
	.align	1
	.globl	func2
	.type	func2, @function
func2:
	addi	sp,sp,-16
	sd	s0,8(sp)
	addi	s0,sp,16
	li	a5,1
	mv	a0,a5
	ld	s0,8(sp)
	addi	sp,sp,16
	jr	ra
	.size	func2, .-func2
	.align	1
	.globl	func1
	.type	func1, @function
func1:
	addi	sp,sp,-16
	sd	ra,8(sp)
	sd	s0,0(sp)
	addi	s0,sp,16
	call	func2
	mv	a5,a0
	mv	a0,a5
	ld	ra,8(sp)
	ld	s0,0(sp)
	addi	sp,sp,16
	jr	ra
	.size	func1, .-func1
	.ident	"GCC: (GNU) 7.2.0"
```

- main.s

```
	.file	"main.c"
	.option nopic
	.text
	.align	1
	.globl	func2
	.type	func2, @function
func2:
	addi	sp,sp,-16
	sd	s0,8(sp)
	addi	s0,sp,16
	li	a5,1
	mv	a0,a5
	ld	s0,8(sp)
	addi	sp,sp,16
	jr	ra
	.size	func2, .-func2
	.align	1
	.globl	func1
	.type	func1, @function
func1:
	addi	sp,sp,-16
	sd	ra,8(sp)
	sd	s0,0(sp)
	addi	s0,sp,16
	call	func2
	mv	a5,a0
	mv	a0,a5
	ld	ra,8(sp)
	ld	s0,0(sp)
	addi	sp,sp,16
	jr	ra
	.size	func1, .-func1
	.align	1
	.globl	main
	.type	main, @function
main:
	addi	sp,sp,-16
	sd	ra,8(sp)
	sd	s0,0(sp)
	addi	s0,sp,16
	call	func1
	mv	a5,a0
	mv	a0,a5
	ld	ra,8(sp)
	ld	s0,0(sp)
	addi	sp,sp,16
	jr	ra
	.size	main, .-main
	.ident	"GCC: (GNU) 7.2.0"
```



### as - GNU Assembler Command

as工具的输入是汇编代码，输出是目标文件。输出目标文件作为ld的输入，最后生成可执行文件。
目标文件是ELF格式。

```
llvm@llvm-vm:~/workspace/gem5/tests/test-progs/test-gnubinutils$ ../../../../freedom/rocket-chip/riscv-tools/riscv-tc-new/bin/riscv64-unknown-linux-gnu-as main.s -o main.o
llvm@llvm-vm:~/workspace/gem5/tests/test-progs/test-gnubinutils$ file main.o 
main.o: ELF 64-bit LSB relocatable, UCB RISC-V, version 1 (SYSV), not stripped
```

### ld – GNU Linker Command


### ar – GNU Archive Command


### nm – List Object File Symbols


### objcopy – Copy and Translate Object Files


### objdump – Display Object File Information


### size – List Section Size and Toal Size

size可以显示出目标文件各个段section的大小。

```
llvm@llvm-vm:~/workspace/gem5/tests/test-progs/test-gnubinutils$ ../../../../freedom/rocket-chip/riscv-tools/riscv-tc-new/bin/riscv64-unknown-linux-gnu-size main.o
   text	   data	    bss	    dec	    hex	filename
    128	      0	      0	    128	     80	main.o
```

### strings – Display Printable Characters from a File

strings列出目标文件中可以打印的符号，默认只显示.data段，-a选项可以显示所有段中的符好。

```
llvm@llvm-vm:~/workspace/gem5/tests/test-progs/test-gnubinutils$ ../../../../freedom/rocket-chip/riscv-tools/riscv-tc-new/bin/riscv64-unknown-linux-gnu-strings -a main.o
GCC: (GNU) 7.2.0
main.c
func2
func1
main
.symtab
.strtab
.shstrtab
.rela.text
.data
.bss
.comment
```

### strip – Discard Symbols from Object File

strip可以去掉目标文件中的可打印符号，减小目标文件的大小，提高执行速度。

```
llvm@llvm-vm:~/workspace/gem5/tests/test-progs/test-gnubinutils$ ../../../../freedom/rocket-chip/riscv-tools/riscv-tc-new/bin/riscv64-unknown-linux-gnu-objdump -t main.o

main.o:     file format elf64-littleriscv

SYMBOL TABLE:
0000000000000000 l    df *ABS*	0000000000000000 main.c
0000000000000000 l    d  .text	0000000000000000 .text
0000000000000000 l    d  .data	0000000000000000 .data
0000000000000000 l    d  .bss	0000000000000000 .bss
0000000000000000 l    d  .comment	0000000000000000 .comment
0000000000000000 g     F .text	0000000000000020 func2
0000000000000020 g     F .text	0000000000000030 func1
0000000000000050 g     F .text	0000000000000030 main

llvm@llvm-vm:~/workspace/gem5/tests/test-progs/test-gnubinutils$ ../../../../freedom/rocket-chip/riscv-tools/riscv-tc-new/bin/riscv64-unknown-linux-gnu-strip main.o
llvm@llvm-vm:~/workspace/gem5/tests/test-progs/test-gnubinutils$ ../../../../freedom/rocket-chip/riscv-tools/riscv-tc-new/bin/riscv64-unknown-linux-gnu-objdump -t main.o

main.o:     file format elf64-littleriscv

SYMBOL TABLE:
no symbols

```

### c++filt – Demangle Command


### addr2line – Convert Address to Filename and Numbers


### readelf – Display ELF File Info






