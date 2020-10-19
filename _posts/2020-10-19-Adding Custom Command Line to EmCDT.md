---
layout: post
title: Adding Custom Command Line to EmCDT
date: 2020-10-19
updated: 2020-10-19
categories: Tech
tags: [Essay,Tech]
description: EmbCDT是目前针对RISC-V指令集架构做的比较完备的开源IDE工具，本文介绍如何增加自定义的编译命令，并让用户通过按钮进行选择。
---

1.如果我们有两个功能一样的库文件，暂且命名为```libabc```和```libabc_00```，一个放在固化在芯片内部，一个在编译的时候添加进去，用户可以根据需要自己选择使用哪个。

```
riscv32-haawking-elf-g++ -march=rv32imc -mabi=ilp32 -mcmodel=medany -mno-save-restore -labc_00
```

![Adding Custom Command Line](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/custom_command_line_to_emcdt.png)


2.定义命令，```abclib```

在文件eclipse-plugins.git\plugins\ilg.gnumcueclipse.managedbuild.cross.riscv\plugin.xml中```RISC-V Targets common options```下面增加该命令的定义。

并设置好按钮选择与否的命令行。

```
            <option
				command="-labc"
				commandFalse="-labc_00"
				defaultValue="false"
				id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.base.target.abclib"
				isAbstract="true"
				name="Use BootRom abclib"
				tip="Use BootRom Library or Your Own Library (-labc)."
				valueType="boolean">
			</option>
```

同时，在同一个文件中，在```%toolchain.name```下，增加类的声明。

```
            <option
				category="ilg.gnumcueclipse.managedbuild.cross.riscv.optionCategory.target"
				id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.saverestore"
				superClass="ilg.gnumcueclipse.managedbuild.cross.riscv.option.base.target.saverestore">
			</option>
```

3.修改plugin.properties文件

在```eclipse-plugins.git\plugins\ilg.gnumcueclipse.managedbuild.cross.riscv\plugin.properties```中，添加如下说明

```
option.target.abclib=BootRom Lib (-labc)

```

4.修改option类

在```Option```增加

```
public static final String OPTION_TARGET_ABCLIB = OPTION_TARGET + "abclib";
```

在```getToolChainFlags```中添加

```
        sValue = getOptionBooleanCommand(config, OPTION_TARGET_ABCLIB);
		if (sValue != null && sValue.length() > 0) {
			sReturn += " " + sValue;
		}
```

到这一步的时候，EmCDT的Makefile中应该就可以看到```-labc```选项了，但是呢，我们并没有在模板工程中添加相应的lib文件，这样编译的时候是会报错，提示找不到相应的lib文件的。

5.在模板工程中添加lib文件

在```eclipse-plugins.git\plugins\ilg.gnumcueclipse.templates.haawking\templates\haawking_exe_c_project\template.xml```中创建lib文件夹并添加lib文件。

```
<process type="org.eclipse.cdt.core.CreateSourceFolder">
			<simple
				name="projectName"
				value="$(projectName)" />
			<simple
				name="path"
				value="xpacks/haawking-devices/lib" />
		</process>
		<process type="ilg.gnumcueclipse.templates.core.ConditionalCopyFolders">
			<simple
				name="projectName"
				value="$(projectName)" />
			<simple
				name="condition"
				value="" />
			<complex-array name="folders">
				<element>
					<simple
						name="source"
						value="xpacks/haawking-devices/lib" />
					<simple
						name="target"
						value="xpacks/haawking-devices/lib" />
					<simple
						name="pattern"
						value=".*[.](a)" />
					<simple
						name="replaceable"
						value="true" />
				</element>
			</complex-array>
		</process>
```


```
<process type="ilg.gnumcueclipse.templates.core.ConditionalAddFiles">
			<simple
				name="projectName"
				value="$(projectName)" />
			<simple
				name="condition"
				value="" />
			<complex-array name="files">
				<element>
					<simple
						name="source"
						value="xpacks/haawking-devices/lib/miz702n/libabc.a" />
					<simple
						name="target"
						value="xpacks/haawking-devices/lib/miz702n/libabc.a" />
					<simple
						name="replaceable"
						value="true" />
				</element>
				<element>
					<simple
						name="source"
						value="xpacks/haawking-devices/lib/miz702n/libabc_00.a" />
					<simple
						name="target"
						value="xpacks/haawking-devices/lib/miz702n/libabc_00.a" />
					<simple
						name="replaceable"
						value="true" />
				</element>
			</complex-array>
		</process>
```

6.创建了lib文件夹并拷贝了lib文件，还需要添加lib文件的路径

添加路径的时候，一定选择```AppendToMBSStringListOptionValues```，否则，将会替代掉之前的变量。

```
<process
			type="org.eclipse.cdt.managedbuilder.core.AppendToMBSStringListOptionValues">
			<simple
				name="projectName"
				value="$(projectName)" />
			<complex-array name="resourcePaths">
				<element>
					<simple
						name="id"
						value="ilg.gnumcueclipse.managedbuild.cross.riscv.option.*.linker.paths" />
					<simple-array name="values">
						<element value="&quot;../xpacks/haawking-devices/lib/miz702n&quot;" />
					</simple-array>
					<simple
						name="path"
						value="" />
				</element>
			</complex-array>
		</process>
```

![Adding Custom Command Line](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/custom_command_line_to_emcdt_lib_path.png)

这个时候，如果lib文件没有问题，那么就可以完成调用自定义库的编译和链接了。

（2020-10-19，希格玛公寓，北京）