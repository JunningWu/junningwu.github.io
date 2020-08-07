---
layout: post
title: SCI (UART) Boot Test Based on FPGA VP Board
date: 2020-08-07
updated: 2020-08-07
categories: Tech
tags: [Essay,Tech]
description: 最近一直在忙着项目，真是进度压死人。一看上次更新博文还是5月份的事情。今天在FPGA板上面调试通过了SCI（也就是串口）的Boot过程，设备启动之后可以跟上位机进行收发数据。回想起来，一步一步把编译器、设备驱动库、链接文件、集成开发环境、启动程序等，一切都是从头从零开始，本来看似稀松平常的一件事，难度比做之前预计的大得多。为自己和团队成员感到骄傲和自豪！加油，Strive For Greadness！！！
---

目前基于risc-v架构，设计一款MCU应该不是天大的难事，至少开源平台上面就有很多成熟的方案，比如pulp和lowrisc，按照官方的教程，一步步操作，至少在FPGA板上进行原型验证，应该不是难事。

难的就是，如果不是采用RISC-V的标准指令集，比如RV32IMC，也不用pulp提供的pulp-sdk，甚至也不用SiFive提供的E studio集成开发环境，而是想自己针对某一领域进行优化，增加自定义的指令，有一套自己独有的芯片启动方式和BootLoader程序，甚至IDE也想自己来尝试一下，那难度就可大多了。

而，这就是我们这边选择的一条路径，所有的都是拥抱开源，站在巨人的肩膀之上。编译器基于LLVM，IDE基于CodeBlocks，串口程序是微软的，处理器指令集架构是RISC-V。

今天这篇帖子呢，主要是介绍SCI，也就是串口UART的boot过程，所需要的的boottable，以及跟上位机交互的方式，后续对于整个芯片还是再进行介绍，这里就简单对芯片支持的部分启动方式做一下说明。

如下图所示，如果在未接调试器（低电平）的时候，则会根据 GPIO34 和 GPIO37 判断是哪种 Boot 方式。如果是 Flash Boot，则会直接跳转到 Flash 地址 0x7DFFE8 处，执行一组跳转指令，跳转指令共两条；这也是芯片通常的启动方式。如果是 GetMode，则会根据 OTPKey （0x7A3BF8）和 OTPBMode （0x7A3BFA）来判断采用哪种 boot 方式；这一点在后续的文章中会再介绍。

今天介绍的SCI Boot方式呢，就需要GPIO34接高电平，GPIO37接地，未接入调试器。芯片中内置的BootLoader，将会根据这两个pin脚的信号电平，选择SCI boot并跳转到SCI_Boot()函数处执行。SCI_Boot()函数，在完成SCI模块的初始化和复用IO的配置以后，就会等待上位机发送波特率检测的约定信息，这里是“A”，如果没有接收到，就一直等着，同时为了防止看门狗复位芯片，这里已经将其关闭。

![Boot Configuration](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/boot_config.png)

好了，Boot过程呢，就先说到这里，下面说一下用户的应用程序，也就是我们要下载到芯片中的程序。SCI Bootloader呢，会将通过串口接收到的应用程序，存放在芯片的L1存储区中，在Boot完成以后，也是从L1的起始地址处开始执行。在进入到应用程序以前，Bootloader的代码，存放在BOOTROM中。

有了基于CodeBlocks的IDE之后，编辑编译程序都比较方便了。编译器是基于LLVM开发的，支持自定义的指令。代码编写好后，直接点击Build，就可以生成SCI Boot所需要的boottable了，有二进制文件，也有文本文件。通过属性信息，可以看到boottable的大小是698字节。这一点很重要，多一个或者少一个字节，程序下载到芯片中，都是不能运行的。

![Program](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/program.png)

![Bin Boottable](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/bin_boottable.png)

接下来就是接上FPGA开发板，打开Win10自带的串口调试助手，这个软件比较好用的一点，就是可以发送文件。

![Host Software](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/host_software.png)

当前版本的SCI BootLoader程序，在进行完模块初始化和复用IO配置以后，会等待上位机发送波特率检测需要的特殊字符，也就是“A”，16进制就是0x41。并且，芯片在接收到该字符完成波特率检测以后，会回传一下接收到的字符。

![Auto Baud Detection](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/autobaud_det.png)

为了保证检测的正确性和有效性，SCI BootLoader呢又额外增加了一次传输，上位机需要再次发送一个任意字符，如果收到的字符与发送的一样，说明波特率没问题，就可以正常通信了。

![One More Auto Baud Detection](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/one_more_test.png)

那么，接下来，我们就可以通过串口调试助手，把应用程序加载到芯片中了。找到刚才编译生成的bin文件，并且点击发送。

![Load Boottable](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/load_boottable.png)

![Send Boottable](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/send_boottable.png)

传输完成以后，BootLoader会自动跳转到应用程序的入口地址处，执行应用程序。我们这里循环打印的是“HAAWKING By CASIA”。

![Run Test](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/sci_boot/run_test.png)

至此，SCI Boot的过程，就简单地介绍了一下。

回想起来，一步一步把编译器、设备驱动库、链接文件、集成开发环境、启动程序等，一切都是从头从零开始，本来看似稀松平常的一件事，难度比做之前预计的大得多。为自己和团队成员感到骄傲和自豪！加油，Strive For Greadness！！！

十年前，我们在做替代芯片的时候，只要芯片造出来，国外厂家的IDE、编译器、调试器等，所有相关的配套，都可以拿来用。五六年前，在做ARM芯片的时候呢，也有ARM强大的生态来获取帮助。几乎都没有把软件工作放在心上，但是，这两年，自己开始做RISC-V芯片之后，发现我们国家更缺芯片配套的软件。

记得十年前上学那会，国科大的杨力祥说，国内做操作系统开发的凤毛麟角，而当年国内做自主芯片编译器开发的，估计也就只有龙芯的冯晓兵老师组吧。

我辈当自强。其实，自己本没有什么大的宏伟目标，只是希望能在对得起学校和研究所的培养之上，能够做出一点有用的贡献；不求惊天动地，但求微末寸功。

（2020-08-07，希格玛公寓，北京）