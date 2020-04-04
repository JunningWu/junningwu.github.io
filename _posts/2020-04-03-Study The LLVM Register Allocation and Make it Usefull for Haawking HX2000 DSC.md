---
layout: post
title: Study The LLVM Register Allocation and Make it Usefull for Haawking HX2000 DSC
date: 2020-04-03
categories: Tech
tags: [Essay,Tech]
description: 本篇主要记录一下初学使用LLVM的寄存器分配Register Allocation，并对HX2000 DSC自定义指令中对寄存器有特殊需求的进行优化实现。
---

寄存器分配问题在于将可以使用无限数量的虚拟寄存器的程序Pv映射到包含有限数量（可能很小）的物理寄存器的程序Pp。每个目标体系结构都有不同数量的物理寄存器。如果物理寄存器的数量不足以容纳所有虚拟寄存器，则其中的一些虚拟寄存器将必须映射到内存中（成为Spilling）。这些虚拟称为溢出虚拟。

## How registers are represented in LLVM

在LLVM中，物理寄存器由通常在1到1023之间的整数表示。要查看如何为特定体系结构定义此编号，您可以阅读该体系结构的GenRegisterNames.inc文件。 例如，通过检查lib/Target/X86/X86GenRegisterInfo.inc，我们看到32位寄存器EAX用43表示，MMX寄存器MM0映射到65。

一些架构包含共享相同物理位置的寄存器。一个著名的例子是X86平台。例如，在X86体系结构中，寄存器EAX，AX和AL共享前八个位。这些物理寄存器在LLVM中被标记为别名。给定特定的体系结构，您可以通过检查其RegisterInfo.td文件来检查哪些寄存器是别名的。 此外，类MCRegAliasIterator枚举所有别名为寄存器的物理寄存器。

RISC-V标准的寄存器共32个整形寄存器，分别编号如下：

```
let RegAltNameIndices = [ABIRegAltName] in {
  def X0  : RISCVReg<0, "x0", ["zero"]>, DwarfRegNum<[0]>;
  let CostPerUse = 1 in {
  def X1  : RISCVReg<1, "x1", ["ra"]>, DwarfRegNum<[1]>;
  def X2  : RISCVReg<2, "x2", ["sp"]>, DwarfRegNum<[2]>;
  def X3  : RISCVReg<3, "x3", ["gp"]>, DwarfRegNum<[3]>;
  def X4  : RISCVReg<4, "x4", ["tp"]>, DwarfRegNum<[4]>;
  def X5  : RISCVReg<5, "x5", ["t0"]>, DwarfRegNum<[5]>;
  def X6  : RISCVReg<6, "x6", ["t1"]>, DwarfRegNum<[6]>;
  def X7  : RISCVReg<7, "x7", ["t2"]>, DwarfRegNum<[7]>;
  }
  def X8  : RISCVReg<8, "x8", ["s0", "fp"]>, DwarfRegNum<[8]>;
  def X9  : RISCVReg<9, "x9", ["s1"]>, DwarfRegNum<[9]>;
  def X10 : RISCVReg<10,"x10", ["a0"]>, DwarfRegNum<[10]>;
  def X11 : RISCVReg<11,"x11", ["a1"]>, DwarfRegNum<[11]>;
  def X12 : RISCVReg<12,"x12", ["a2"]>, DwarfRegNum<[12]>;
  def X13 : RISCVReg<13,"x13", ["a3"]>, DwarfRegNum<[13]>;
  def X14 : RISCVReg<14,"x14", ["a4"]>, DwarfRegNum<[14]>;
  def X15 : RISCVReg<15,"x15", ["a5"]>, DwarfRegNum<[15]>;
  let CostPerUse = 1 in {
  def X16 : RISCVReg<16,"x16", ["a6"]>, DwarfRegNum<[16]>;
  def X17 : RISCVReg<17,"x17", ["a7"]>, DwarfRegNum<[17]>;
  def X18 : RISCVReg<18,"x18", ["s2"]>, DwarfRegNum<[18]>;
  def X19 : RISCVReg<19,"x19", ["s3"]>, DwarfRegNum<[19]>;
  def X20 : RISCVReg<20,"x20", ["s4"]>, DwarfRegNum<[20]>;
  def X21 : RISCVReg<21,"x21", ["s5"]>, DwarfRegNum<[21]>;
  def X22 : RISCVReg<22,"x22", ["s6"]>, DwarfRegNum<[22]>;
  def X23 : RISCVReg<23,"x23", ["s7"]>, DwarfRegNum<[23]>;
  def X24 : RISCVReg<24,"x24", ["s8"]>, DwarfRegNum<[24]>;
  def X25 : RISCVReg<25,"x25", ["s9"]>, DwarfRegNum<[25]>;
  def X26 : RISCVReg<26,"x26", ["s10"]>, DwarfRegNum<[26]>;
  def X27 : RISCVReg<27,"x27", ["s11"]>, DwarfRegNum<[27]>;
  def X28 : RISCVReg<28,"x28", ["t3"]>, DwarfRegNum<[28]>;
  def X29 : RISCVReg<29,"x29", ["t4"]>, DwarfRegNum<[29]>;
  def X30 : RISCVReg<30,"x30", ["t5"]>, DwarfRegNum<[30]>;
  def X31 : RISCVReg<31,"x31", ["t6"]>, DwarfRegNum<[31]>;
  }
}
```

LLVM中的物理寄存器按寄存器类分组。相同寄存器类中的元素在功能上等效，并且可以互换使用。每个虚拟寄存器只能映射到特定类别的物理寄存器。例如，在X86体系结构中，某些虚拟机只能分配给8位寄存器。寄存器类由TargetRegisterClass对象描述。

有时，主要用于调试目的，更改目标体系结构中可用的物理寄存器的数量很有用。这必须在TargetRegisterInfo.td文件中静态完成。只是RegisterClass的grep，其最后一个参数是寄存器列表。只是注释掉一些是避免使用它们的一种简单方法。一种更合理的方式是从分配顺序中明确排除某些寄存器。有关此示例，请参见lib/Target/X86/X86RegisterInfo.td中GR8寄存器类的定义以及lib/Target/RISCV/RISCVRegisterInfo.td中GPRNoX0寄存器类的定义。

虚拟寄存器也用整数表示。与物理寄存器相反，不同的虚拟寄存器从不共享相同的编号。物理寄存器是在TargetRegisterInfo.td文件中静态定义的，并且不能由应用程序开发人员创建，而虚拟寄存器则不是这种情况。为了创建新的虚拟寄存器，使用方法MachineRegisterInfo::createVirtualRegister（）。此方法将返回一个新的虚拟寄存器。使用IndexedMap <Foo，VirtReg2IndexFunctor>可以保存每个虚拟寄存器的信息。

在寄存器分配之前，指令的操作数主要是虚拟寄存器，尽管也可以使用物理寄存器。为了检查给定的机器操作数是否为寄存器，可以使用布尔函数MachineOperand::isRegister()。要获取寄存器的整数代码，请使用MachineOperand::getReg()。指令可以定义或使用寄存器。例如，ADD reg:1026 := reg:1025 reg:1024定义了寄存器1024，并使用了寄存器1025和1026。给定一个寄存器操作数，方法MachineOperand::isUse()通知指令是否正在使用该寄存器。方法MachineOperand::isDef()通知是否正在定义该寄存器。

在完成寄存器分配之前，LLVM BitCode中的物理寄存器，称为** 预着色(Pre-color)寄存器**。预着色的寄存器可用于许多不同的情况，例如，传递函数调用的参数以及存储特定指令的结果。有两种类型的预着色寄存器：隐式定义的寄存器和显式定义的寄存器。显式定义的寄存器是常规操作数，可以使用MachineInstr::getOperand(int)::getReg()进行访问。为了检查指令隐式定义了哪些寄存器，请使用TargetInstrInfo::get(opcode)::ImplicitDefs，其中opcode是目标指令的操作码。显式和隐式物理寄存器之间的一个重要区别是，后者是为每条指令静态定义的，而前者可能会根据所编译的程序而有所不同。例如，代表函数调用的指令将始终隐式定义或使用同一组物理寄存器。要读取指令隐式使用的寄存器，请使用TargetInstrInfo::get(opcode)::ImplicitUses。预先着色的寄存器对任何寄存器分配算法都施加了约束。寄存器分配器必须确保在它们还处于活动状态时，它们都不会被虚拟寄存器的值覆盖。

## Mapping virtual registers to physical registers

有两种方法可以将虚拟寄存器映射到物理寄存器（或内存插槽memory slot）。第一种方法，我们称为直接映射，是基于对TargetRegisterInfo和MachineOperand类的方法的使用。第二种方法，我们将称为间接映射，它依赖于VirtRegMap类来插入load和store指令与内存进行数据交互。

直接映射为寄存器分配器的开发人员提供了更大的灵活性。但是，它更容易出错，并且需要更多的实现工作。基本上，程序员将必须指定在正在编译的目标函数中应在何处插入load和store指令，以便在内存中获取和存储值。要将物理寄存器分配给给定操作数中存在的虚拟寄存器，请使用MachineOperand::setReg(p_reg)。要插入存储指令，请使用TargetInstrInfo::storeRegToStackSlot(...)，要插入加载指令，请使用TargetInstrInfo::loadRegFromStackSlot。

间接映射使应用程序开发人员免受插入load和store指令的复杂性的影响。为了将虚拟寄存器映射到物理寄存器，请使用VirtRegMap::assignVirt2Phys(vreg，preg)。为了将某个虚拟寄存器映射到内存，请使用VirtRegMap::assignVirt2StackSlot(vreg)。此方法将返回vreg值所在的堆栈插槽(stack slot)。如果有必要将另一个虚拟寄存器映射到同一堆栈插槽，请使用VirtRegMap::assignVirt2StackSlot(vreg，stack_location)。使用间接映射时要考虑的重要一点是，即使虚拟寄存器映射到内存，仍然需要将其映射到物理寄存器。该物理寄存器是应该在存储之前或重新加载之后在其中找到虚拟寄存器的位置。

如果使用了间接策略，则在将所有虚拟寄存器映射到物理寄存器或堆栈插槽后，有必要使用溢出对象(Spilling Object)在代码中放置加载和存储指令。每个已映射到堆栈插槽的虚拟机将在定义后存储到内存中，并在使用前加载。溢出程序(Spilling )的实现尝试回收加载/存储指令，避免不必要的指令。有关如何调用溢出程序的示例，请参见lib/CodeGen/RegAllocLinearScan.cpp中的RegAllocLinearScan::runOnMachineFunction。

## Handling two address instructions

除极少数例外情况（例如函数调用）外，LLVM机器代码指令是三个地址指令。即，每个指令预期最多定义一个寄存器，并最多使用两个寄存器。但是，某些体系结构使用两个地址指令。在这种情况下，定义的寄存器也是使用的寄存器之一。 例如，在X86中，诸如ADD %EAX，%EBX之类的指令实际上等效于%EAX =%EAX +%EBX。

为了产生正确的代码，LLVM必须将代表两个地址指令的三个地址指令转换为真正的两个地址指令。LLVM为此目的提供了传递TwoAddressInstructionPass。必须在分配寄存器之前运行它。执行之后，生成的代码可能不再是SSA形式。 例如，在诸如%a = ADD %b %c之类的指令被转换为两个指令，例如：

```
%a = MOVE %b
%a = ADD %a %c
```

注意，在内部，第二条指令表示为ADD %a[def/use] %c。 即，指令同时使用和定义了寄存器操作数%a。

在寄存器分配期间发生的重要转换称为SSA解构阶段(** SSA Deconstruction Phase **)。SSA表格简化了对程序的控制流程图执行的许多分析。但是，传统的指令集不实现PHI指令。因此，为了生成可执行代码，编译器必须将PHI指令替换为其他保留其语义的指令。

可以通过多种方式安全地从目标代码中删除PHI指令。最传统的PHI解构算法将PHI指令替换为复制指令。这就是LLVM采用的策略。SSA解构算法在lib/CodeGen/PHIElimination.cpp中实现。为了调用此过程，必须在寄存器分配器的代码中将标识符PHIEliminationID标记为必需。

## Built in register allocators

LLVM基础结构为应用程序开发人员提供了三种不同的寄存器分配器：

--快速-此寄存器分配器是调试版本的默认设置。它在基本块级别上分配寄存器，尝试将值保留在寄存器中并适当地重用寄存器。

--基本—这是注册分配的一种增量方法。实时范围按启发式驱动顺序一次分配给一个寄存器。由于可以在分配期间动态重写代码，因此该框架允许将有趣的分配器开发为扩展。它本身不是生产寄存器分配器，而是潜在的有用的独立模式，用于对错误进行分类并作为性能基准。

--贪婪-默认分配器。这是基本分配器的高度优化的实现，该实现合并了全局活动范围分割。该分配器努力工作以最小化溢出代码的成本。

--PBQP —基于分区布尔二次编程（PBQP）的寄存器分配器。该分配器的工作原理是构造一个表示正在考虑的寄存器分配问题的PBQP问题，使用PBQP求解器解决该问题，并将该解决方案映射回寄存器分配。


