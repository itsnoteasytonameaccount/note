---
title: linux常用指令记录(2)
tags: linux
date: 2018/04/10 18:23:09
---

方便下次用起来找
<!-- more -->

# 内容

## 进程相关操作

### 1.ps
说实话不熟这个指令,只知道-u参数是显示用户名,-e全部程序
一般配合grep筛选
```bash
# 例,查找进程信息中包含python的行
ps -aux|grep python
# 其余可自行查看帮助
ps --help
ps --help a
# 配合kill pid使用可以关闭想关闭的进程
```

### 2.netstat
查看端口监听

|参数|功能|
|:-|:-|
|-a|显示全部端口监听|
|-p|显示pid,可执行文件路径|
|-t|筛选tcp进程|
|-e|显示额外信息（进程所属用户）|
|-u|筛选udp进程|
|-n|直接显示端口号而不是将其转换为对应程序的名字|

```bash
# 例:显示所有tcp端口监听程序，并筛选其中名字带有nginx的,显示端口号所有用户和pid
netstat -napte|grep nginx
```

## 安装程序相关

### 1.tar
解压程序

|参数|功能|
|:-|:-|
|-v|列出输出的文件|
|-x|解压|
|-f|使用文件名|
|-z|gz格式|

tar可选项居多，我了解的就这几个

### 2.wget

从远程路径下载文件

linux安装程序的途径之一就是用wget远程下载源码,
解压后在源文件根目录运行configure文件,设置安装选项,然后，makefile
make install就好了
也算是通用途径，当然，源文件也不是一定要通过wget下载

其他的安装方法在不同版本系统应该还是有区别的
像ubuntu安装是用的apt-get命令
centos系统则是yum直接安装，或用下载rpm安装包，再用rpm命令更新或安装

## 磁盘操作

### 1.fdisk
fdisk这个命令了解的也不多
只知道fdisk -l显示空间信息和挂载磁盘信息

### 2.mount
同样知之甚少，只知道如下
```bash
# 例将某磁盘挂载在某一目录下
mount /dev/adb1 /
```

## 其他

### 1.反向引号
执行嵌套指令
```bash
# 例，删除当前目录所有名字以php开头的
rm -rf `ls|grep ^php`
# 先执行反向引号中的命令,列出名字带有php的,再删除
```
### 2.众多的shell指令
用shell执行脚本来进行批处理

### 3.service
这个命令我在centos上就没用成功过，每次都提示什么systemctl什么的
换用systemctl也还是失败
基本格式
service reload serviceName
service start serviceName
service stop servicecName

### 4.ln
这个命令之前忘了，在此补充
ln [options] from to

|选项|作用|
|:-|:-|
|-s|软连接，to的路径里存的是from的路径，类似快捷方式|
|-P|硬链接,不填options时默认情况为此|

硬链接相当于复制一份文件，但所有连接的文件占的空间只有一份，并且修改其中之一其余都会修改，删除其中之一，其余仍然存在，相当于链表中三个节点的指针指向同一文件，记得操作系统中学过文件信息似乎就是用链表存的

### 结语
有错误请指出
