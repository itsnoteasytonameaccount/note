---
title: 注册表
date: 2018/01/14 18:10:34
tags:
- windows
- 注册表
---
# 注册表

## 引言
对于写惯了代码的人来说，命令操作要比图形界面方便不少，并且学习中不少配置工作是没有图形操作程序的，因而在windows系统中命令行也是时常要用的功能，然而windows每次打开命令行一般是win+R后输入cmd，还要cd转到路径,十分不便。<!--more-->

## 右键菜单修改

此处参考了[这篇百度经验](https://jingyan.baidu.com/article/6c67b1d6ea3fd02787bb1eea.html)

### 1.打开注册表编辑器
win+r打开运行，输入regedit，运行注册表编辑器。
找到
>计算机\HKEY_CLASSES_ROOT\Directory\

### 2.修改内容
进入directory后可看到四个子目录（如图background,defaulticon,shell,sehllex)。

![图片无法显示](../image/1.png)

上面那篇百度经验的指示是
>在shell目录下新建项CommandPrompt，并为CommandPrompt添加子项Command并设置commad数值为
```bash
cmd.exe /k cd "%1"
```
就全部搞定了

## 拓展

### 1.shell目录
通过注册表目录结构结构得出
>shell目录下每项代表右键菜单下一项，该项的默认值代表该项在右键菜单下的名称（经测试，如果默认值为空，该项的右键菜单菜单项名为该项的名称），该项下的command子项的值代表该项要执行的指令

添加其他程序的右键执行也是同理，在shell目录下添加自定义子项，在自定义子项下添加command子项，把command项的值改为对应的可执行程序的路径即可(或要执行的指令)，如改为D://英雄联盟/lol.exe(可以不加.exe后缀)

### 2.shellex目录
未知

### 3.background目录
我们知道在文件管理器中有三种右键菜单，右击空白部分,右击文件，右击文件夹，经测试，background文件夹决定的是右击空白时的右键菜单，上面提到的百度经验指导添加的是右击文件夹与文件时的右键菜单。
background目录下也有两个子目录shell,shellex,添加项目的方式同上。

### 4.项目图标
在右键菜单中看到有的项目有图标，如图，

![图片无法显示](../image/7.png)

若想要添加，照葫芦画瓢，找到图中有图标的右键菜单项对应的注册表目录，如图git_gui,发现该项还有一个Icon数据，值为图标路径

![图片无法显示](../image/5.png)

找到要添加图标的菜单项对应的注册表项，如图，为其添加数据Icon,值为你准备好的ico路径

![图片无法显示](../image/4.png)

最终效果如图

![图片无法显示](../image/6.jpg)

### 5.用。。。打开
文件上右击菜单项添加
路径位于
```bash
HKEY_CLASSES_ROOT\*\shell\
```
添加方法同上述，但要注意此处添加的命令不同

### 6.注册表项总结
[正好搜各个路径的作用时搜到这篇](https://blog.csdn.net/liyun123gx/article/details/22199151)

__不同的注册表根路径__

|根键|描述|
|:-|:-|
|HKEY_CLASSES_ROOT  |当前`classes`配置，对应的是`HKEY_LOCAL_MACHINE\Software\Classes`、`HKEY_CURRENT_USER\Software\Classes`，内容为前述两个`Classes`合并，修改`HKEY_LOCAL_MACHINE\Software\Classes`、`HKEY_CURRENT_USER\Software\Classes`，`HKEY_CLASSES_ROOT`的内容也会被改变，修改`HKEY_CLASSES_ROOT`会改变`HKEY_LOCAL_MACHINE\Software\Classes`|
|HKEY_CURRENT_USER  |当前用户配置，修改后作用范围是当前用户，修改时同时被更新到`HKEY_USERS`对应的分支下|
|HKEY_LOCAL_MACHINE |机器配置，内容结构和`HKEY_CURRENT_USER`相同，作用范围是当前机器|
|HKEY_USERS         |所有用户的预配置，用代表用户的一串id作为分支，每个分支的内容结构与`HKEY_CURRENT_USER`内容结构相同，若修改的是当前用户的分支，修改会被立即更新到`HKEY_CURRENT_USER`。`HKEY_USERS\.Default`是新用户默认配置，`HKEY_USERS\{id}`是用户的配置，`HKEY_USERS\{id}_classes`是`HKEY_USERS\{id}\Software\Classes`的内容拷贝|
|HKEY_CURRENT_CONFIG|见上文引用的外链|

__常用注册表路径__

下述列表为`HKEY_LOCAL_MACHINE`中的路径，可类推到其他根路径下，对当前用户生效，或对其他用户生效。

|路径|作用|
|:-|:-|
|`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Advanced\Folder`|文件夹选项|
|`HKEY_LOCAL_MACHINE\SOFTWARE\Classes\*\shell\`|文件的右键菜单|
|`HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Directory\shell\`|文件夹的右键菜单|
|`HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Directory\background\shell\`|文件资源管理器空白部分的右键菜单|
|`HKEY_CURRENT_USER\Volatile Environment`|用户变量|
|`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`|本机变量|

待补充

## 结语

如有错误欢迎指教