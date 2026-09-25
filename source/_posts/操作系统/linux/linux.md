---
title: linux常用指令记录(1)
tags: linux
date: 2018/04/07 15:51:07
---
## 前言
寒假的时候租了台服务器，国内的，也算是学习到了linux的一些指令的用法了，由于最近事多许久没用，快忘的差不多了，趁全忘光前赶紧边回忆边记录下来
<!--more-->
## 基础文件操作
### 1.mv
格式
```bash
mv options source directory
```
|参数|功能|
|:-|:-|
|-f|冲突时强制替换|
*其他参数都不怎么了解*


mv可以使用的场合有：重命名文件文件夹，相较于rename我更习惯用mv来重命名，移动文件或文件夹
例
```bash
//将a重命名为b
mv /a /b
//将a移动至b
mv /a /b/
```

### 2.mkdir
我只知道是创建文件夹的
```bash
mkdir dirName
```

### 3.rm
格式
```bash
rm option file
```
|参数|功能|
|:-|:-|
|-f|强制删除无需确认|
|-r|删除文件夹及其子文件|
|-d|删除空文件夹|

### 4.rename
rename要注意的是需要传入三个必须的参数
```bash
# replacement新名字
# file文件名
# 匹配文件名需要被替代部分的的正则表达式
rename expressions repaclement file
# 例如
rename name newname.txt name.txt
# 重命名结果为newname.txt.txt
rename * newname.txt name.txt
# 重命名结果为newname.txt
```

### 5.ls
列出当前目录下文件
格式
```bash
ls option dir
```
|参数|功能|
|:-|:-|
|-a|列出隐藏文件在内的所有文件|
*才疏学浅，只了解这一个参数*

一般配合grep命令(文本搜索有关的都可以配合grep)
```bash
# 例找出当前文件夹名字带有python的
ls|grep python
```
### 6.find
```bash
# 只知道一个,找出a文件夹下，名字带b的文件
find /a/ -name b
# 或更慢的
find /a/ -name *|grep b
```
### 7.cat
显示文本内容
```bash
# 将文本内容输出至控制台
cat a.txt
# 将a文本内容输出至b
cat a.html >> b.html
```

### 8.whereis
显示文件所在位置
这个具体什么作用不清楚，我一般用来找安装的可执行程序
```bash
# 例
whereis python
```

### 9.cp
复制文件
```bash
cp soucefile direct
# cp的选项我只知道-f是强制替换
```


## 基础用户操作
### 1.useradd
作用添加用户

|参数|功能|
|:-|:-|
|-s|指定用户shell,具体不清楚，只是之前配置sftp用户时在网上查据说用于禁止用户ssh登陆，设-s /bin/false|
|-g|决定用户所在的组|
|-p|决定用户密码|
|-d|家目录，用户的根目录|

以上配置也可在创建用户后用usermod添加或更改
密码还可以用passwd

删除用户userdel userName

### 2.groupadd
添加组
groupadd groupName
groups查看组中所有用户
groups userName查看用户所属的组名

删除组groupdel groupname

### 3.chown
更改目录所属用户
chown option user(group) file(dir)
