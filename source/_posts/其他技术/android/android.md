---
title: android开发初学遇问题及解决方案
tag:
- android
- android studio
date: 2019/03/01
---

最近学android碰到的问题
<!-- more -->

# 版本问题

项目报错,Error:No toolchains found in the NDK toolchains folder for ABI with prefix: mips64el-linux-android,原因似乎是新建项目的gradle版本号不对，修改就行,另一个也是版本号不对引起的错，（重新同步一下试试，很多问题都可以重新同步一下试试）

Error:(16, 0) Minimum supported Gradle version is 4.1. Current version is 2.14.1
改版本号，或者在android studio中自动更新配置就行（同步一下试试）

# 虚拟化的支持问题导致avd无法启动
emulator: ERROR: x86 emulation currently requires hardware acceleration!
网上介绍都是要安装haxm，[如此处](https://blog.csdn.net/sihai12345/article/details/72868635).

但一般用android studio都会自动装好，一般不存在android studio上显示已安装，实际却未安装的情况，如果以前成功启动过avd，也不可能是bios中关了vt-x支持的问题(必须吐槽神舟电脑上bios许多选项都被锁了根本没有vt-x选项)。

可能碰到的问题不大相同，但解决方案基本是相同的，主要是自己尝试。

本人尝试步骤如下

1.  先查cpu是否支持vt-x，参数中显示支持（仔细找,都会写的），不支持的话我也不知道怎么办
2.  以前已装过haxm，先在android sdk目录下的extras/intal文件夹找到安装文件卸载并重装，此时你电脑上若存在导致不支持虚拟化的问题，是会报错并且无法安装的（此处获得报错1）。
3.  上网查找发现虚拟化技术和vmware也有关,正好电脑上有vmware，打开vmware中的虚拟机，果然也报错与某服务冲突(此处获得报错2，上网查找该服务与hype-v有关).
4.  在命令行中用emulator命令（emulator在androidsdk中，此处我是配置过环境变量的）启动avd，除了上文提到的报错还有一句报错大意是“检查你的某某服务是否安装了”（此处获得报错三，依然是和hype-v有关)
5.  通过多次尝试大致明了问题出在hype-v,百度hype-v发现其似乎与虚拟化技术冲突，进入打开或关闭windows功能关闭hype-v，再启动虚拟机，一切正常。

# 网络问题

Download https://services.gradle.org/distributions/gradle-4.6-all.zip

这个错是文件下载失败，翻墙重试或[去网上下需要的版本](http://services.gradle.org/distributions/)，放在相应目录下,目录自己找

# 命令行打开emulator问题

