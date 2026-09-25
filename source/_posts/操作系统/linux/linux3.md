---
title: centos安装配置mysql
tags:
- linux
- mysql
date: 2018/07/14
---

临时新租了服务器，但安mysql时出了些错，做个记录
<!-- more -->

# 内容

## 安装工具简介
yum：centos中类似apt-get的命令，依赖python环境
yum用法(以下为复制粘贴，不保证正确性)

>1.使用YUM查找软件包 命令：yum search
>2.列出所有可安装的软件包 命令：yum list
>3.列出所有可更新的软件包 命令：yum list updates
>4.列出所有已安装的软件包 命令：yum list installed
>5.列出所有已安装但不在 Yum Repository 内的软件包  命令：yum list extras
>6.列出所指定的软件包 命令：yum list
>7.使用YUM获取软件包信息 命令：yum info
>8.列出所有软件包的信息 命令：yum info
>9.列出所有可更新的软件包信息 命令：yum info updates
>10.列出所有已安装的软件包信息 命令：yum info installed
>11.列出所有已安装但不在 Yum Repository 内的软件包信息  命令：yum info extras
>12.列出软件包提供哪些文件 命令：yum provides

两个常用的命令
yum install package
yum uninstall package

rpm为另一个安装工具，需要先下载rpm的安装包
常用命令
``bash
rpm -qa
//查找
rpm -ivh
//-i install,-v 似乎是列出输出文件，-h hash什么的，不明确
```
## mysql安装配置
mysql需要安装3项
devel,client,server分别用于支持开发，连接，提供服务
我的安装方法是

### 1.安装yum的mysql仓库(下载源)
>这一步不知道是否必要，你可以先用yum install mysql-devel*试试看是否存在安装包
mysql仓库rpm安装包[在这下](https://dev.mysql.com/downloads/repo/yum/)
获取下载连接后可用wegt命令直接获取安装包
若无wget
```bash
yum install wget
```
获取安装包后
**rpm -ivh 包名**来安装
之后就可以用
```bash
yum install mysql-community-client mysql-community-server mysql-community-devel
```
来安装mysql了

安装好后mysql的配置文件路径为/etc/my.cnf
一般采取默认设置就行。
### 配置步骤
```bash
#会在默认配置的datadir初始化
#若datadir中已存在文件会报错
#可在log-error路径中查看报错输出，默认是/var/log/mysqld.log
#error-log设置查看配置文件(如果配了)
mysqld --initialize
#--initialize步骤还会生成默认密码，可在log-error路径中查看
#windows上初始密码在datadir的"***.err"文件中
#安装服务(windows上才需要)
mysqld --install
#后台启动
#最好确认配置文件中有usr=mysql，避免以root用户运行
mysqld &
#用initial的密码登陆
mysql -uroot -p你的密码
#登录后要用命令修改用户密码，否则会报错
alter user 'root'@'localhost' identified by '你的密码';
#mysql以前的password()函数和password字段都没了，只能用上诉命令修改密码
```
若忘记密码则可重启mysqld并一跳过验证（管理员），方式如下
```bash
mysqld --skip-grant-tables
```
跳过验证后不可用idetified方式修改密码
```bash
alter user 'root'@'localhost' identified by '你的密码';#报错
```
其他方案
先将密码置空
```bash
update user set authentication_string='' where user='root';
```
重启mysqld,再验证登陆修改密码
```bash
mysql -uroot -p
alter user 'root'@'localhost' identified by '你的密码'；
```
