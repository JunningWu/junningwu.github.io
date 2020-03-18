---
layout: post
title: 如何针对RISC-V添加自定义指令？
date: 2019-11-28
categories: Tech
tags: [Eaasy,Tech]
description: 如何针对RISC-V添加自定义指令？GNU的开发者Jim Wilson在邮件列表中的回复，比较具有启发意义。
---
It is easier to add instructions to binutils than gcc.  For binutils, 
we have the .insn support that Kito Cheng added that lets you specify 
each instruction field individually, and can be used for custom 
instructions.  If you want something more programmer friendly, then 
you can add a line to opcodes/riscv-opc.c to add an instruction.  Just 
copy a similar instruction and modify the fields as appropriate.  You 
can find the match/mask patterns in include/opcode/riscv-opc.h.  For 
the operand letters, you can look at the code in gas/config/tc-riscv.c 
riscv_ip() that handles them, e.g. search for 'p' to find the p 
support, and note that the first one is for compressed support 'Cp' 
and the second one is the plain 'p'.  Adding instructions this way 
will require some understanding of how binutils works, but the 
assemble/disassemble stuff is pretty easy.  It is the linker stuff 
that is complicated.  If you have your own instruction formats, and 
need new relocations and/or relaxations, then that can get very 
complicated very quickly, and I'm not going to try to explain that 
here.  The simulator is also pretty easy, though we have two of them, 
the gdb simulator which is not upstream, and the QEMU simulator which 
is upstream.  Both should be pretty easy to modify.  Oh, and I suppose 
we also have spike, but I haven't looked at that one much. 

There is a binutils tool called cgen that lets you construct an 
assembler from an architecture description file.  This is an easier 
way to go if you want to do a lot of architecture experimenting, but 
it is not how the current assembler is written, and changing to a new 
assembler design at this point would likely be painful.  Embecosm 
incidentally has a cgen risc-v assembler port.  I don't know if they 
plan to release the sources for it, and I don't know how many existing 
RISC-V binutils features are supported in it. 

For gcc, the first question is what do you mean by gcc support.  Are 
you OK using an extended asm to hand code in the instruction?  That is 
trivial.  Do you want an intrinsic that will generate the instruction 
for you?  This is not very hard.  Do you want the compiler optimizer 
to automatically generate the instruction?  This is harder.  You need 
to add a pattern to the gcc/config/riscv/riscv.md file to describe the 
instruction.  If the instruction is performing a common operation, 
then just adding the instruction pattern may be enough to get it 
generated.  You will have to spend some timing debugging the compiler 
to get the pattern details right so that it gets generated when 
appropriate, but this is generally not too hard.  If the instruction 
is performing a less common operation, then you may have to do work on 
target independent and/or target dependent optimization passes to get 
the instruction to be generated, and this will require a lot of gcc 
internals knowledge, and possibly a lot of time. 

There are a number of GCC internals tutorials that have been written 
by various people over the years.  We have a link to some of them on 
the gcc web site. 
    https://gcc.gnu.org/wiki/GettingStarted#Tutorials.2C_HOWTOs 
GCC internals is always changing as development progresses, so some of 
the info in these will be out-of-date.  And of course there are lots 
of text books that talk about compiler design and implementation if 
you need a general introduction to compilers. 

For binutils, it is a much smaller development community than gcc, and 
the core developers tend to stay with it longer, so there is less 
tutorial type info available.  There is one on the web site 
    https://sourceware.org/binutils/binutils-porting-guide.txt 
but it appears to be a brief high level description and maybe not very 
useful to you.  There aren't many textbooks that cover what binutils 
does, but the linker part is the only part worthy of a textbook.  For 
that, I would suggest "Linkers and Loaders" by John R Levine.  I 
haven't actually read this, but I know a number of the people 
mentioned in the Acknowledgements section and have heard good things 
about the book. 

Both binutils and gcc have mailing lists where you can ask questions 
if you are serious about getting involved in development, and need 
help understanding something.  Usually the best way to get started is 
to just pick a bug report or enhancement request, start reading 
sources, try various ways to fix or implement it until you find 
solution, then asking on the mailing lists if you have a good 
solution, and iterate until it is right, learning how the sources work 
along the way.  Then pick another one and repeat for a few years until 
you are an expert. 

https://groups.google.com/a/groups.riscv.org/forum/#!msg/sw-dev/sL_OHXYj3LY/Gsm6sBc9BQAJ