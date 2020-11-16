---
layout: post
title: Adding Flash Password Configuration to Haawking IDE
date: 2020-11-14
updated: 2020-11-14
categories: Tech
tags: [Essay,Tech]
description: Haawking的DSC芯片，内置128KB的Flash或者256KB的Flash，加密模块CSM所需的密码，就存放再Flash中，为了给使用者提供解密的便利，需要在Haawking IDE中增加CSMKEY修改的界面和功能。
---

# 最终效果
Haaking IDE V0.0.7版本中，首次支持CSMKEY修改功能，在调试界面，用户可以向Password的输入框中输入密码，每个32位，共4个输入框，128位。

![Haawking IDE V0.0.7](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/flash-password-console.png)

使用者只需要向输入框中输入所需的密码，Haawking IDE就会调用底层的Openocd工具，完成CSMKEY写入的工作。

## 增加四个Flash Password的配置选项和默认值。

在文件```eclipse-plugins.git\plugins\ilg.gnumcueclipse.debug.gdbjtag.openocd\src\ilg\gnumcueclipse\debug\gdbjtag\openocd\ConfigurationAttributes.java```中，添加配置选项，如下所示：
```
public static final String GDB_SERVER_FLASH_PASSWORD_NUM1 = PREFIX + ".gdbServerFlashPasswordNum1"; //$NON-NLS-1$
public static final String GDB_SERVER_FLASH_PASSWORD_NUM2 = PREFIX + ".gdbServerFlashPasswordNum2"; //$NON-NLS-1$
public static final String GDB_SERVER_FLASH_PASSWORD_NUM3 = PREFIX + ".gdbServerFlashPasswordNum3"; //$NON-NLS-1$
public static final String GDB_SERVER_FLASH_PASSWORD_NUM4 = PREFIX + ".gdbServerFlashPasswordNum4"; //$NON-NLS-1$
```

在文件```eclipse-plugins.git\plugins\ilg.gnumcueclipse.debug.gdbjtag.openocd\src\ilg\gnumcueclipse\debug\gdbjtag\openocd\preferences\DefaultPreferences.java```中，添加默认值，这里密码的默认值为```全1```，如下所示：

```
public static final String GDB_SERVER_FLASH_PASSWORD_NUM1_DEFAULT = "FFFFFFFF";
public static final String GDB_SERVER_FLASH_PASSWORD_NUM2_DEFAULT = "FFFFFFFF";
public static final String GDB_SERVER_FLASH_PASSWORD_NUM3_DEFAULT = "FFFFFFFF";
public static final String GDB_SERVER_FLASH_PASSWORD_NUM4_DEFAULT = "FFFFFFFF";
```

## 修改Haawking IDE界面显示的标签和说明注释

在文件```eclipse-plugins.git\plugins\ilg.gnumcueclipse.debug.gdbjtag.openocd\src\ilg\gnumcueclipse\debug\gdbjtag\openocd\ui\messages.properties```文件中，增加Haawking IDE界面显示的标签和说明注释，修改及效果如下：

```
DebuggerTab.gdbServerFlashPasswordNum1_Label=Password #1:
DebuggerTab.gdbServerFlashPasswordNum1_ToolTipText=\
The 1st Password In Flash, \n\
should be in Hex Format,\n\
(default FFFFFFFF).

DebuggerTab.gdbServerFlashPasswordNum2_Label=Password #2:
DebuggerTab.gdbServerFlashPasswordNum2_ToolTipText=\
The 2nd Password In Flash, \n\
should be in Hex Format,\n\
(default FFFFFFFF).

DebuggerTab.gdbServerFlashPasswordNum3_Label=Password #3:
DebuggerTab.gdbServerFlashPasswordNum3_ToolTipText=\
The 3rd Password In Flash, \n\
should be in Hex Format,\n\
(default FFFFFFFF).

DebuggerTab.gdbServerFlashPasswordNum4_Label=Password #4:
DebuggerTab.gdbServerFlashPasswordNum4_ToolTipText=\
The 4th Password In Flash, \n\
should be in Hex Format,\n\
(default FFFFFFFF).
```

![Haawking IDE V0.0.7](https://github.com/JunningWu/junningwu.github.io/raw/master/_posts/pics/flash-password-lable.png)

## 修改调试界面的主文件

以下操作，均在文件```eclipse-plugins.git\plugins\ilg.gnumcueclipse.debug.gdbjtag.openocd\src\ilg\gnumcueclipse\debug\gdbjtag\openocd\ui\TabDebugger.java```中完成。

### 增加四个与Flash Password相关的句柄声明

```
private Text fGdbServerFlashPasswordNum1;
private Text fGdbServerFlashPasswordNum2;
private Text fGdbServerFlashPasswordNum3;
private Text fGdbServerFlashPasswordNum4;
```

### 在createGdbServerGroup中，添加输入框

```
		{
			Label label = new Label(comp, SWT.NONE);
			label.setText(Messages.getString("DebuggerTab.gdbServerFlashPasswordNum1_Label"));
			label.setToolTipText(Messages.getString("DebuggerTab.gdbServerFlashPasswordNum1_ToolTipText"));

			fGdbServerFlashPasswordNum1 = new Text(comp, SWT.SINGLE | SWT.BORDER);
			GridData gd = new GridData();
			gd.widthHint = 60;
			gd.horizontalSpan = ((GridLayout) comp.getLayout()).numColumns - 1;
			fGdbServerFlashPasswordNum1.setLayoutData(gd);
		}
		
		{
			Label label = new Label(comp, SWT.NONE);
			label.setText(Messages.getString("DebuggerTab.gdbServerFlashPasswordNum2_Label"));
			label.setToolTipText(Messages.getString("DebuggerTab.gdbServerFlashPasswordNum2_ToolTipText"));

			fGdbServerFlashPasswordNum2 = new Text(comp, SWT.SINGLE | SWT.BORDER);
			GridData gd = new GridData();
			gd.widthHint = 60;
			gd.horizontalSpan = ((GridLayout) comp.getLayout()).numColumns - 1;
			fGdbServerFlashPasswordNum2.setLayoutData(gd);
		}
		
		{
			Label label = new Label(comp, SWT.NONE);
			label.setText(Messages.getString("DebuggerTab.gdbServerFlashPasswordNum3_Label"));
			label.setToolTipText(Messages.getString("DebuggerTab.gdbServerFlashPasswordNum3_ToolTipText"));

			fGdbServerFlashPasswordNum3 = new Text(comp, SWT.SINGLE | SWT.BORDER);
			GridData gd = new GridData();
			gd.widthHint = 60;
			gd.horizontalSpan = ((GridLayout) comp.getLayout()).numColumns - 1;
			fGdbServerFlashPasswordNum3.setLayoutData(gd);
		}
		
		{
			Label label = new Label(comp, SWT.NONE);
			label.setText(Messages.getString("DebuggerTab.gdbServerFlashPasswordNum4_Label"));
			label.setToolTipText(Messages.getString("DebuggerTab.gdbServerFlashPasswordNum4_ToolTipText"));

			fGdbServerFlashPasswordNum4 = new Text(comp, SWT.SINGLE | SWT.BORDER);
			GridData gd = new GridData();
			gd.widthHint = 60;
			gd.horizontalSpan = ((GridLayout) comp.getLayout()).numColumns - 1;
			fGdbServerFlashPasswordNum4.setLayoutData(gd);
		}
```

### 添加修改功能

```
		fGdbServerFlashPasswordNum1.addModifyListener(new ModifyListener() {
			@Override
			public void modifyText(ModifyEvent e) {

				// make the target port the same
				//fTargetPortNumber.setText(fGdbServerFlashPasswordNum1.getText());
				scheduleUpdateJob();
			}
		});
		
		fGdbServerFlashPasswordNum2.addModifyListener(new ModifyListener() {
			@Override
			public void modifyText(ModifyEvent e) {

				// make the target port the same
				//fTargetPortNumber.setText(fGdbServerFlashPasswordNum2.getText());
				scheduleUpdateJob();
			}
		});
		
		fGdbServerFlashPasswordNum3.addModifyListener(new ModifyListener() {
			@Override
			public void modifyText(ModifyEvent e) {

				// make the target port the same
				//fTargetPortNumber.setText(fGdbServerFlashPasswordNum3.getText());
				scheduleUpdateJob();
			}
		});
		
		fGdbServerFlashPasswordNum4.addModifyListener(new ModifyListener() {
			@Override
			public void modifyText(ModifyEvent e) {

				// make the target port the same
				//fTargetPortNumber.setText(fGdbServerFlashPasswordNum4.getText());
				scheduleUpdateJob();
			}
		});
```

### 在doStartGdbServerChanged中使能

```
		fGdbServerFlashPasswordNum1.setEnabled(enabled);
		fGdbServerFlashPasswordNum2.setEnabled(enabled);
		fGdbServerFlashPasswordNum3.setEnabled(enabled);
		fGdbServerFlashPasswordNum4.setEnabled(enabled);
```

### 在initializeFrom中完成初始值的设定

```
				//Flash Password 
                fGdbServerFlashPasswordNum1.setText(
						configuration.getAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM1,
								DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM1_DEFAULT));
				fGdbServerFlashPasswordNum2.setText(
						configuration.getAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM2,
								DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM2_DEFAULT));
				fGdbServerFlashPasswordNum3.setText(
						configuration.getAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM3,
								DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM3_DEFAULT));
				fGdbServerFlashPasswordNum4.setText(
						configuration.getAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM4,
								DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM4_DEFAULT));
```

以及```Default值```的设定。

```
           // Flash Password
			fGdbServerFlashPasswordNum1.setText(DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM1_DEFAULT);
			fGdbServerFlashPasswordNum2.setText(DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM2_DEFAULT);
			fGdbServerFlashPasswordNum3.setText(DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM3_DEFAULT);
			fGdbServerFlashPasswordNum4.setText(DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM4_DEFAULT);
```

### 在isValid中判断一下输入是否有效

```
			if (fGdbServerFlashPasswordNum1 != null && fGdbServerFlashPasswordNum1.getText().trim().isEmpty()) {
				setErrorMessage("Flash Password Num1?");
				result = false;
			}
			
			if (fGdbServerFlashPasswordNum2 != null && fGdbServerFlashPasswordNum2.getText().trim().isEmpty()) {
				setErrorMessage("Flash Password Num2?");
				result = false;
			}
			
			if (fGdbServerFlashPasswordNum3 != null && fGdbServerFlashPasswordNum3.getText().trim().isEmpty()) {
				setErrorMessage("Flash Password Num3?");
				result = false;
			}
			
			if (fGdbServerFlashPasswordNum4 != null && fGdbServerFlashPasswordNum4.getText().trim().isEmpty()) {
				setErrorMessage("Flash Password Num4?");
				result = false;
			}
```

### 在canSave中判断是否能够写入

```
        if (fGdbServerFlashPasswordNum1 != null && fGdbServerFlashPasswordNum1.getText().trim().isEmpty())
				return false;
		if (fGdbServerFlashPasswordNum2 != null && fGdbServerFlashPasswordNum2.getText().trim().isEmpty())
				return false;
		if (fGdbServerFlashPasswordNum3 != null && fGdbServerFlashPasswordNum3.getText().trim().isEmpty())
				return false;
		if (fGdbServerFlashPasswordNum4 != null && fGdbServerFlashPasswordNum4.getText().trim().isEmpty())
				return false;
```

### 在performApply中执行写入操作

```
			//Flash Password
			if (!fGdbServerFlashPasswordNum1.getText().trim().isEmpty()) {
				String password = fGdbServerFlashPasswordNum1.getText().trim();
				configuration.setAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM1, password);
			} else {
				Activator.log("empty fGdbServerFlashPasswordNum1");
			}
			if (!fGdbServerFlashPasswordNum2.getText().trim().isEmpty()) {
				String password = fGdbServerFlashPasswordNum2.getText().trim();
				configuration.setAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM2, password);
			} else {
				Activator.log("empty fGdbServerFlashPasswordNum2");
			}
			if (!fGdbServerFlashPasswordNum3.getText().trim().isEmpty()) {
				String password = fGdbServerFlashPasswordNum3.getText().trim();
				configuration.setAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM3, password);
			} else {
				Activator.log("empty fGdbServerFlashPasswordNum3");
			}
			if (!fGdbServerFlashPasswordNum4.getText().trim().isEmpty()) {
				String password = fGdbServerFlashPasswordNum4.getText().trim();
				configuration.setAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM4, password);
			} else {
				Activator.log("empty fGdbServerFlashPasswordNum4");
			}
```

### 在setDefaults中，设置默认值

```
			configuration.setAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM1,
					DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM1_DEFAULT);
			configuration.setAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM2,
					DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM2_DEFAULT);
			configuration.setAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM3,
					DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM3_DEFAULT);
			configuration.setAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM4,
					DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM4_DEFAULT);
```

## 修改OpenOCD的命令行读取功能

在```eclipse-plugins.git\plugins\ilg.gnumcueclipse.debug.gdbjtag.openocd\src\ilg\gnumcueclipse\debug\gdbjtag\openocd\Configuration.java```中，添加-u命令，让OpenOCD支持密码写入命令。

```
            lst.add("-u");
			lst.add(" " + configuration.getAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM1,
					DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM1_DEFAULT) + "." + 
					configuration.getAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM2,
					DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM2_DEFAULT) + "." + 
					configuration.getAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM3,
					DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM3_DEFAULT) + "." + 
					configuration.getAttribute(ConfigurationAttributes.GDB_SERVER_FLASH_PASSWORD_NUM4,
					DefaultPreferences.GDB_SERVER_FLASH_PASSWORD_NUM4_DEFAULT));
```


最后，通过上述修改，Haawking IDE已经能够调用OpenOCD发出CSMKEY修改的命令，具体效果，还得等调试以及芯片回来以后再确定。

（2020-11-14，希格玛公寓，北京）