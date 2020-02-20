---
layout: post
title: GNU Binutils目录框架及简介
date: 2020-02-20
categories: 技术文章
tags: [技术文章]
description: 最近因为工作需要，对于自定义的指令进行支持，感觉写.word或者.insn对于单条指令的验证和测试还算ok，但是如果指令数量较多，则就不太合适，还是需要工具能够进行汇编生成可执行代码的。而这部分工作需要修改的就是GNU Binutils。
---

GNU Binutils源代码的大部分位于下列这几个目录中。binutils的某些组件是库，可在内部以及其他项目中使用。例如，BFD库用于GNU GDB调试器。这些库具有自己的顶级目录。GNU Binutils的目录框架如下：
                         binutils
                            |
                            |
    ----------------------------------------------------------
   |       |     |       |      |       |   |     |     |     |
   |       |     |       |      |       |   |     |     |     |
 include  bfd  opcodes  cpu  binutils  gas  ld  gprof  gold  elfcpp
