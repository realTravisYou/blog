---
title: 使用Logadd2在codesys中打印日志
description: ""
date: 2026-07-10T07:06:07.094Z
preview: ""
draft: false
tags: ['PLC']
categories: ['电控']
---

# 如何在codesys中记录日志
>前提：确保在Library Manager中已经添加了**CmpLog**和**Component Manager**这两个系统库。  
在 CODESYS 中，你需要先创建一个功能块（Function Block），然后再在这个功能块下右键添加对应的方法（Method）。    

## `一、为了可以在主程序中可以调用，不必每次写完整的方法，我将完整的 FB_SystemLogger 拆分为主功能块和方法两部分。`  
### 1. 主功能块：FB_SystemLogger  
在工程左侧的 POUs 树中，右键添加一个 POU，类型选择 Function Block，语言选择 ST，命名为 `FB_SystemLogger`。  
#### 变量声明区 (Declaration)  
```Structured Text
FUNCTION_BLOCK FB_SystemLogger
VAR_INPUT
    // 允许外部传入自定义的组件名称，默认值为 'DefaultApp'
    sComponentName : STRING(80) := 'DefaultApp'; 
END_VAR
VAR
    udiCmpId : UDINT;
    bInit : BOOL := FALSE;
END_VAR
```
#### 代码实现区 (Implementation)
这里负责在功能块第一次被调用时，向系统注册组件名称并获取 ID。
```Structured Text
// 初始化：注册自定义组件名称（每个实例只执行一次）
IF NOT bInit THEN
    Component_Manager.CMAddComponent2(sComponentName, 16#00000001, ADR(udiCmpId), 0);
    bInit := TRUE;
END_IF
```
### 2.方法一：`LogInfo` (记录常规信息) 
右键点击你刚建好的 `FB_SystemLogger`，选择 Add Object -> Method，命名为 `LogInfo`，返回类型写 BOOL。
#### 变量声明区
```Structured Text
METHOD LogInfo : BOOL
VAR_INPUT
    sMessage : STRING(255); // 要写入的日志文本
END_VAR
```
#### 代码实现区
```Structured Text
IF bInit THEN
    CmpLog.LogAdd2(
        hLogger := CmpLog.LogConstants.LOG_STD_LOGGER, // 伪句柄：指向标准 PLC 日志
        udiCmpID := udiCmpId,                          // 刚才注册的组件ID
        udiClassID := CmpLog.LogClass.LOG_INFO,        // 级别：INFO (蓝色图标)
        udiErrorID := 0,                               // 错误代码 (无错误填0)
        udiInfoID := 0,                                // 多语言文本ID (不用填0)
        pszInfo := sMessage                            // 你的自定义日志文本
    );
    LogInfo := TRUE;
ELSE
    LogInfo := FALSE;
END_IF
```
### 3. 方法二：LogError (记录错误信息)
再次右键点击 `FB_SystemLogger`，添加方法，命名为 `LogError`，返回类型 `BOOL`。
#### 变量声明区
```Structured Text
METHOD LogError : BOOL
VAR_INPUT
    sMessage : STRING(255);
    udiErrorCode : UDINT := 16#00000001; // 错误代码，可以根据具体业务传入不同代号
END_VAR
```
#### 代码实现区
```Structured Text
IF bInit THEN
    CmpLog.LogAdd2(
        hLogger := CmpLog.LogConstants.LOG_STD_LOGGER,
        udiCmpID := udiCmpId,
        udiClassID := CmpLog.LogClass.LOG_ERROR,       // 级别：ERROR (红色图标)
        udiErrorID := udiErrorCode,                    
        udiInfoID := 0,
        pszInfo := sMessage
    );
    LogError := TRUE;
ELSE
    LogError := FALSE;
END_IF
```
### 4. 方法三：`LogWarning` (记录警告信息)
可选择再添加一个警告级别的方法，命名为 `LogWarning`，返回类型 `BOOL`。
#### 变量声明区
```Structured Text
METHOD LogWarning : BOOL
VAR_INPUT
    sMessage : STRING(255);
END_VAR
```
#### 代码实现区
```Structured Text
IF bInit THEN
    CmpLog.LogAdd2(
        hLogger := CmpLog.LogConstants.LOG_STD_LOGGER,
        udiCmpID := udiCmpId,
        udiClassID := CmpLog.LogClass.LOG_WARNING,     // 级别：WARNING (黄色图标)
        udiErrorID := 0,                    
        udiInfoID := 0,
        pszInfo := sMessage
    );
    LogWarning := TRUE;
ELSE
    LogWarning := FALSE;
END_IF
```
完成以上步骤后，这个类似于 C++ 类的 `FB_SystemLogger` 就彻底封装好了，你可以在任何子功能块或主程序中独立实例化并使用了。
## 二、调用示例
#### 变量声明区
```Structured Text
PROGRAM PLC_PRG
VAR
	// 实例化日志对象，并在声明时传入专属名称（类似于 C++ 的带参构造函数）
    Logger : FB_SystemLogger;
	bInfo    : BOOL;
    bError   : BOOL;
	bWarn	 : BOOL;
END_VAR
```
#### 代码实现区
```Structured Text
// 1. 传入专属名字并调用初始化
Logger(sComponentName := 'myLogComponent');

// 2. 触发日志
IF bInfo THEN
	Logger.LogInfo(sMessage := 'hello world');
	bInfo := FALSE;
END_IF
// 3. 触发错误
IF bError THEN
	Logger.LogError(sMessage := 'There are some Errors');
	bError := FALSE; 
END_IF
// 4. 触发警告
IF bWarn THEN
	Logger.LogWarning(sMessage := 'Warning, Warning, Warning');
	bWarn := FALSE;
END_IF
```
日志记录查看如图所示。
![在codesys中日志查看界面](.\images\codesysLogInfoPage.png)

