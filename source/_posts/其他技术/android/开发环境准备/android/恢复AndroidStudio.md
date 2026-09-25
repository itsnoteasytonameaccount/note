---
title: 新装的系统恢复Android Studio配置
tags:
- 环境
- android
---

重新安装系统后，需要安装android studio，有很多东西需要恢复，主要配置如下

# GRADLE_USER_HOME

需要在设置中指定gradle对应的位置，如不指定的话默认会将`.gradle`安装在`<系统盘>\<用户>\<当前用户名>`目录下，也就是用户home目录

![image-20250622214524361](C:\Home\WorkSpace\Document\Markdown\source\_posts\开发环境准备\android\assets\image-20250622214524361.png)

或者配置环境变量`GRADLE_USER_HOME`

# JAVA_HOME

如下界面可设置gradle用到的jdk位置

![image-20250622213923687](C:\Home\WorkSpace\Document\Markdown\source\_posts\开发环境准备\android\assets\image-20250622213923687.png)



# ANDROID_HOME

如下界面设置android sdk路径

![image-20250622200247521](.\assets\image-20250622200247521.png)

或者配置`ANDROID_HOME`变量来指定android sdk的路径

另外若配置了`ANDROID_HOME`会影响到的路径如下

| 路径                            | 意义                   | 备注                                                     |
| :------------------------------ | ---------------------- | -------------------------------------------------------- |
| `%ANDROID_HOME%/platform-tools` | adb工具等的路径        |                                                          |
| `%ANDROID_HOME%/emulator`       | android模拟器的路径    | 在比较老的android studio版本是`%ANDROID_HOME%/tools`     |
| `%ANDROID_HOME%/cmdline-tools`  | avdmanager等工具的路径 | 在比较老的android studio版本是`%ANDROID_HOME%/tools/bin` |

# ANDROID_USER_HOME

安卓模拟器等配置会存放在这个环境指定的目录

默认值是`%USER_HOME%/.android`，也就是说如果不指定这个变量，`.android`目录会默认安装在用户home目录。

> 注意：比较老的android studio不会识别这个变量，而是默认使用`%USER_HOME%/.android`或者若配置了ANDROID_SDK_HOME变量则使用`%ANDROID_SDK_HOME%/.android`

这个变量还会影响其他路径

| 变量                  | 默认值                       | 备注                                                  |
| --------------------- | ---------------------------- | ----------------------------------------------------- |
| ANDROID_EMULATOR_HOME | `%ANDROID_USER_HOME%`        | 比较老的android studio为`%ANDROID_SDK_HOME%/.android` |
| ANDROID_AVD_HOME      | `%ANDROID_EMULATOR_HOME/avd` | avd配置存放路径                                       |

# 启动前设置环境变量

如果不想修改应用到全局可以写一个bat文件，仅在启动前设置环境变量

示例如下

```bash
@echo off
set GRADLE_USER_HOME=F:\environment\.gradle
set JAVA_HOME=F:\environment\jdk-14.0.2
set ANDROID_HOME=F:\environment\AndroidSDK
set ANDROID_USER_HOME=F:\environment\AndroidSDK\.android
set ANDROID_AVD_HOME=%ANDROID_USER_HOME%\avd
start .\studio64.exe
```

# 仍然存在的问题

目前emulator似乎无法正常启动，两部分原因

1. avd配置文件中写的都是绝对路径，但是avd所在硬盘盘符变了，所以导致没办法正确读取avd配置
2. 似乎是qemu啥的配置也出了问题，只能尝试重新安装或更新一下qemu
