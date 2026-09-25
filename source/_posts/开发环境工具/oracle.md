---
title: oracle中创建数据库
tags: oracle
date: 2018/05/13
---

# 前言

oracle的数据库手动创建比较麻烦，因此记录以下在windows10上的oracle数据库手动创建（命令行）经验，本次创建经验基本依照[此篇博客](https://www.2cto.com/database/201511/449820.html)得到的，因此本文内容很多地方是照搬的，但也有一些地方不同，可以用来对照参考。
<!--more-->

# 安装

## 概念
(以下结论都是我短期学习成果，只具有参考价值)

oracle数据库与以前用过的mysql不同，一个实例与一个数据库是一一对应的，开启客户端程序连接实例就已经连上数据库，不像mysql，连上了服务，还要用use database选择数据库。Oracle与mysql的‘数据库’意义差不多的应该是‘表空间’这一概念

## 步骤

### 修改init文件

修改已有的示例文件中的配置，安装oracle后，会存在示例配置文件，
一般已有的示例文件名是Init.ora之类的，路径不一定，至少我找到Init.ora的路径与网上给出的都不同，最好还是用一下搜索软件，搜索init.ora或initsmpl.ora,initsample.ora之类的文件名
我的电脑中路径是**D:\app\Administrator\product\11.2.0\dbhome_1\dbs\init.ora**

修改后的配置文件应当存放在你的oracle_home路径中，并改名为**init+你的实例sid+.ora**形式的文件名，之后startup数据库时需要用到。
oracle_home的路径也不尽相同，一般结尾都是**products\版本号\db_home或databasehome**
我的电脑中是**D:\app\Administrator\product\11.2.0\dbhome_1**

```bash
# 本人参照网上博客修改后的配置
db_name='mydata'
memory_target=1G
processes = 150
audit_trail ='db'
db_block_size=8192
db_domain=''
db_recovery_file_dest='D:\app\Administrator\product\11.2.0\my_database\flash_recovery_area'
db_recovery_file_dest_size=2G
diagnostic_dest='D:\app\Administrator\product\11.2.0\my_database'
dispatchers='(PROTOCOL=TCP) (SERVICE=ORCLXDB)'
open_cursors=300
remote_login_passwordfile='EXCLUSIVE'
undo_tablespace='UNDOTBS1'
# You may want to ensure that control files are created on separate physical
# devices
control_files = ('D:\app\Administrator\product\11.2.0\my_database\ora_control1.ctl', 'D:\app\Administrator\product\11.2.0\my_database\ora_control2.ctl')
compatible ='11.2.0'
audit_file_dest='D:\app\Administrator\product\11.2.0\my_database\adump'
core_dump_dest='D:\app\Administrator\product\11.2.0\my_database\ddump'
undo_management='AUTO'
```
注意：以上配置文件中你填的路径需要提前创建好，文件不要创建，之后启动数据库时会自动生成
另外在undo_tablespace这一行填的名称需要在create database时用到

### 创建实例

指令格式为
```bash
oradim -new -sid 实例sid -initpwd sys用户的密码
# sid是用来标识实例的，随便起
```
创建后还要设置环境变量ORACLE_SID=实例sid 才行，否则如果你有多个实例，可能默认连接的不是你创建的这个实例

实例只是一个空壳子，能连接但做出的操作保存不了，没什么意义（此句照搬一些博客原话，本人未曾尝试）
就像没通电的插座，能插电器，但是充不了电

### 以管理员身份连接实例并创建二进制文件

```bash
# 以下命令顺序执行
# 不进行登陆开启命令工具
sqlplus /nolog

# sysdba权限登陆
connect sys/sys用户的密码 as sysdba

# 创建二进制文件
# 默认会在你的%oracle_home%\database路径下寻找名为**init+你的实例sid+.ora**的文件
# 并在%oracle_home%\database路径下生成名为**SPFILE+你的实例名+.ora**的文件
# 生成的文件在create database时会用到
# 这一步如果报错请检查是否是放错了init文件的位置
create spfile from pfile;

# 以下命令是以不挂载的形式启动数据库
# 具体什么意思不清楚，似乎是暂时不把数据库挂载在实例上
startup nomount;
```

### 创建数据库

本人创建数据库用的脚本，以sys用户执行，其中所有路径都应视自己计算机情况以及自己需要填写
以下脚本配置数据库配置文件，控制文件以及日志文件的位置
```bash
create database mydata
undo tablespace
UNDOTBS1
# 此处tablespace名称不需要加引号，
# 并且应当与上文中提到的配置文件中undo_tablespace值相同
# ，否则会报错
datafile
    'D:\app\Administrator\product\11.2.0\my_database\system_01.dbf' size 100m
     Autoextend on next 10m maxsize unlimited
sysaux datafile
    'D:\app\Administrator\product\11.2.0\my_database\system_02.dbf' size 100m
     Autoextend on next 10m maxsize unlimited
logfile
group 1 ('D:\app\Administrator\product\11.2.0\my_database\logfile1.rdo') size 10m,
group 2 ('D:\app\Administrator\product\11.2.0\my_database\logfile2.rdo') size 10m
character set utf8;
```
如果报如下错误,请检查你的init文件中是否添加了多余的参数
参考网页[此处](http://www.cnblogs.com/zrcoffee/archive/2012/11/21/2780468.html)
```bash
ORA-32004: obsolete or deprecated parameter(s) specified for RDBMS instance
```
### 执行sql文件

以下四个文件请自行搜索路径

catalog.sql  创建系统常用的数据字典试图和同义词
catproc.sql  建立pl/sql功能的使用环境
pupbld.sql   使用sqlplus环境时需要(system登陆)
scott.sql    创建scott模式

执行方法
```bash
@ 对应sql文件的绝对路径
```

注意：执行第三，四个脚本时需要切换到system用户
用以下命令切换为system用户
```bash
connect system/manager
```
### 连接plsql
1.  安装与oracle数据库位数一致的oracle client和plsql。
2.  先不登陆进入Plsql
3.  进入tools(工具)->preferences(首选项)->connections(连接) 界面
4.  找到你oracle client目录下的oci.dll文件
5.  修改oci library(oci库)路径为你的oci.dll路径
6.  重启plsql,如果数据库名为空，手动填写，默认格式为localhost:1521/你的实例名，再填写对应的用户名密码，连接
7.  如果无法连接，用lsnrctl status查看新建的实例是否已注册到监听服务
8.  若没有注册，尝试以下方法
    (1)重启：
    lsnrctl stop
    lsnrctl start
    (2)修改配置文件listener.ora,再重启
    配置示例
    ```bash
    SID_LIST_LISTENER =
    (SID_LIST =
       (SID_DESC =
         (GLOBAL_DBNAME = MYDATA)
         (SID_NAME = mydatabase)
        )
       (SID_DESC =
         (SID_NAME = mydatabase)
         (ORACLE_HOME = D:\app\Administrator\product\11.2.0\dbhome_3)
         (ENVS = "EXTPROC_DLLS=ONLY:D:\app\Administrator\product\11.2.0\dbhome_3\bin\oraclr11.dll")
       )
    )
    ```
    listener.ora路径可在lsnrctl status中看到

# 后记
感谢上文提到的博客对我学习过程带来的帮助

