---
layout: post
title: Adding Flash or RAM Project Exchange Function to Haawking IDE
date: 2020-11-30
updated: 2020-11-30
categories: Tech
tags: [Essay,Tech]
description: Haawking的DSC芯片，内置大容量的FLASH和RAM，如果代码量不大，则可以选择在RAM中运行，否则，就需要在Flash中运行；为了方便使用者对FLASH和RAM工程进行切换，现在Haawking IDE中增加相应功能。
---

# 最终效果
Haaking IDE V0.0.8版本中，首次支持RAM和FLASH工程进行一键切换的功能。

![Haawking IDE V0.0.8](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/haawking-ide-flash-ram-exchange.png)

使用者在新建工程的时候，可以选择FLASH工程或者RAM工程，而对于已有工程，只需要通过下拉列表，选择同一芯片型号下FLASH或RAM，Haawking IDE就会调用底层逻辑，完成工程属性和相关链接文件以及宏定义的切换工作。

## 增加1个选择框，新建工程的时候选择FLASH或RAM。

在文件```eclipse-plugins.git\plugins\ilg.gnumcueclipse.templates.haawking\templates\haawking_exe_c_project\template.xml```中，添加配置选项，如下所示：

```
<property
			id="HaawkingLink"
			label="RAM/FLASH"
			description="%Haawking.content.description"
			type="select"
			default="RAM"
			hidden="false"
			persist="true">
			<item
				value="RAM"
				label="Program Running in RAM" />
			<item
				value="FLASH"
				label="Program Running in FLASH" />
		</property>
```

这样在新建工程的时候，就会出现一个名为```“RAM/FLASH”```的下拉框，选择项为```"Program Running in RAM"```和```"Program Running in FLASH"```。

![Haawking IDE V0.0.8](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/haawking-ide-flash-ram-select.png)

## 新建工程添加默认连接文件

在文件```eclipse-plugins.git\plugins\ilg.gnumcueclipse.templates.haawking\templates\haawking_exe_c_project\template.xml```中，添加条件语句，如下所示：

使用者在新建工程的时候，可以选择FLASH工程或者RAM工程，如果选择Flash工程，则会默认添加相关的宏定义和连接文件。

这里调用了一个变量```$(deviceName)```，用来区分不同芯片。

```
	<!-- RAM/FLASH options -->
	<if condition="$(HaawkingLink)==FLASH">
		<process type="org.eclipse.cdt.managedbuilder.core.SetMBSStringOptionValue">
			<simple
				name="projectName"
				value="$(projectName)" />
			<complex-array name="resourcePaths">
				<element>
					<simple
						name="id"
						value="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.flashram" />
					<simple
						name="value"
						value="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.flashram.$(deviceName)flash" />
					<simple
						name="path"
						value="" />
				</element>
			</complex-array>
		</process>
	</if>
```


## 修改Haawking IDE工程属性界面，增加FLASH/RAM切换下拉框

在文件```eclipse-plugins.git\plugins\ilg.gnumcueclipse.managedbuild.cross.riscv\plugin.xml```文件中，修改Haawking IDE工程属性界面，增加FLASH/RAM切换下拉框：

```
			<option
			    id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.base.target.flashram"
				isAbstract="true"
				name="Choose FLASH/RAM"
				tip="Choose FLASH/RAM."
				valueType="enumerated">
				<enumeratedOptionValue
					command="DSC28027-RAM"
					id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.flashram.dsc28027ram"
					name="Choose Running in RAM(DSC28027)">
				</enumeratedOptionValue>
				<enumeratedOptionValue
					command="DSC28027-FLASH"
					id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.flashram.dsc28027flash"
					name="Choose Running in FLASH(DSC28027)">
				</enumeratedOptionValue>
				<enumeratedOptionValue
					command="DSC28034-RAM"
					id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.flashram.dsc28034ram"
					name="Choose Running in RAM(DSC28034)">
				</enumeratedOptionValue>
				<enumeratedOptionValue
					command="DSC28034-FLASH"
					id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.flashram.dsc28034flash"
					name="Choose Running in FLASH(DSC28034)">
				</enumeratedOptionValue>
				<enumeratedOptionValue
					command="DSC28335-RAM"
					id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.flashram.dsc28335ram"
					name="Choose Running in RAM(DSC28335)">
				</enumeratedOptionValue>
				<enumeratedOptionValue
					command="DSC28335-FLASH"
					id="ilg.gnumcueclipse.managedbuild.cross.riscv.option.target.flashram.dsc28335flash"
					name="Choose Running in FLASH(DSC28335)">
				</enumeratedOptionValue>
			</option>
```

显示效果最终效果图所示，这里不再重复添加。

## 修改编译选项，能够访问下拉框中的命令

以下操作，均在文件```eclipse-plugins.git\plugins\ilg.gnumcueclipse.managedbuild.cross.riscv\src\ilg\gnumcueclipse\managedbuild\cross\riscv\Option.java```中完成。

### 在public class Option增加flashram选项的声明

```
public static final String OPTION_TARGET_RUN_ENV = OPTION_TARGET + "flashram";
```

### 在public static String getToolChainFlags(IConfiguration config)中，添加参数获取的调用

```
        sValue =  getRiscvTargetRunEnvOption(config);
		if (sValue != null && sValue.length() > 0) {
			sReturn += " " + sValue;
		}
```

### 实现getRiscvTargetRunEnvOption

本实现方式，与TI的CCS一致，用户通过下拉列表，选择不同芯片型号的FLASH或RAM属性，目前支持三款芯片。

```
	private static String getRiscvTargetRunEnvOption(IConfiguration config) {

		String sReturn = "";
		String sValue;

		String sIsa = getOptionEnumCommand(config, OPTION_TARGET_RUN_ENV);
		if (sIsa != null && sIsa.length() > 0) {
			//sReturn += sIsa;
			if("DSC28027-FLASH".equals(sIsa))
			{
				sReturn += "-D__RUNNING_IN_FLASH_  -T DSC28027_link_FLASH.ld"; 
				//OptionUseFLash = true;
			}
			else if("DSC28027-RAM".equals(sIsa))
			{
				sReturn += "-T DSC28027_link_RAM.ld";
				//OptionUseFLash = false;
			}
			else if("DSC28034-FLASH".equals(sIsa))
			{
				sReturn += "-D__RUNNING_IN_FLASH_  -T DSC28034_link_FLASH.ld"; 
				//OptionUseFLash = true;
			}
			else if("DSC28034-RAM".equals(sIsa))
			{
				sReturn += "-T DSC28034_link_RAM.ld";
				//OptionUseFLash = false;
			}
			else if("DSC28335-FLASH".equals(sIsa))
			{
				sReturn += "-D__RUNNING_IN_FLASH_  -T DSC28335_link_FLASH.ld"; 
				//OptionUseFLash = true;
			}
			else if("DSC28335-RAM".equals(sIsa))
			{
				sReturn += "-T DSC28335_link_RAM.ld";
				//OptionUseFLash = false;
			}
			else
			{
				//OptionUseFLash = true;
			}
			
		}
		
		if (sReturn != null) {
			sReturn = sReturn.trim();
		}

		return sReturn;
	}
```



最后，通过上述修改，Haawking IDE已经能够调用OpenOCD发出CSMKEY修改的命令，具体效果，还得等调试以及芯片回来以后再确定。

（2020-11-30，希格玛公寓，北京）