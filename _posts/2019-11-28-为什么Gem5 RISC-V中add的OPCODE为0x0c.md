---
layout: post
title: 为什么Gem5 RISC-V中add的OPCODE为0x0c?
date: 2019-11-28
categories: Tech
tags: [Eaasy,Tech]
description: 为什么Gem5 RISC-V中add的OPCODE为0x0c，在看Gem5 RISC-V的源码的decoder.isa时候，发现add的opcode为0x0c，与手册上面的0x33不一样，但是程序都是可以正常运行的。后来查资料发现了答案。
---

我还是RISC-V和汇编编码的新手。我想要命令的操作码/二进制值。但令我困惑的是A.不同的页面列出了命令的不同操作码和B. 10个命令具有相同的操作码。我怀疑B的aswer是不同的命令描述相同的机制，但我仍然不确定哪些操作码是正确的。
```
来源：https： //github.com/riscv/riscv-opcodes/blob/20e4f0285c563e5a403bd6ba735beadbbd3c203e/opcodes add rd rs1 rs2 16 = 0 15..10 = 0 9..7 = 0 6..2 = 0x0C 1..0 = 3

资料来源： https： //content.riscv.org/wp-content/uploads/2017/05/riscv-spec-v2.2.pdf 0110011 ADD
```
那么为什么说gidub页面说ADD的操作码是0C，十进制是12，而0110011是十进制的51？

前7位表示指令的操作码。github源和pdf都列出了ADD的相同操作码。0x0C = 0000_1100二进制。但github源代表5位（6..2），所以0x0C = 01100二进制。任何有效操作码的前2位始终为11二进制。连接01100 11，你得到0110011二进制，51十进制。

视觉上（使用按位leftshift然后OR）：
```
01100 11 -> ADD Opcode
----- -- 
0x0C  3  -> 0x0C << 2 | 3 -> 12*4 + 3 = 51
```
具有相同操作码的指令表示BEQ和BNE都具有操作码1100011 = BRANCH，将具有另一个字段，其进一步定义指令的功能。因此BRANCH操作码1100011对所有分支指令进行分组。要区分BEQ（分支相等）和BNE（分支不相等），您必须查看funct3字段。BEQ有funct3 = 000，BNE有funct3 = 001。funct3字段将唯一标识BRANCH（1100011）指令的功能：BEQ，BNE，BLT，BGE，BLTU，BGEU）。

像LUI这样的一些指令由操作码唯一标识，因此不需要funct3字段。其他指令如OP（0110011）操作码，需要一个funct3字段和一个funct7字段。注意ADD和SUB都具有相同的操作码（OP = 0110011）以及相同的f​​unct3（000），因此funct7字段是区分器。ADD的funct7是0000000而SUB的funct7是0100000。
```
https://content.riscv.org/wp-content/uploads/2017/05/riscv-spec-v2.2.pdf
```
第19章的开头显示了一个包含所有有效OPCODES及其二进制值的表。请记住，每个OPCODE的前2位是11，表中省略了这一点。例如，查看表找到STORE，如果我们想知道STORE的操作码，则向左扫描到[6：5]列，然后找到01.从STORE扫描到inst [4：2]行，你找到000。由此我们可以构造STORE操作码01 000 11 - > 0100011.所有STORE指令（SB，SH，SW）将0100011作为其操作码。要确定指令是SB，SH还是SW，请查看funct3字段000 = SB，001 = SH，010 = SW。