---
title: subversion配置
date: 2018/11/24
tags:
- subversion
---

# 配置文件
<!-- more -->
## svnserve.conf

分三个配置文件，一个是svnserve.conf，svnserve daemon的配置文件，配置svnserve daemon的选项(This file controls the configuration of the svnserve daemon, if you use it to allow access to this repository.  (If you only allow access through http: and/or file: URLs, then this file is irrelevant..))。

了解的配置

```bash
[general]
# 权限可选read,write,none
# 匿名访问者只有读权限
anon-access = read
# 通过验证登录的访问者有读写权限
auth-access = write
# 用户配置文件的位置
password-db = passwd
# 权限配置文件的位置
authz-db = authz
# 实例描述
realm = My First Repository
```
## authz

权限配置,分为单版本库和多版本库

```bash
# 别名，具体作用不知，下为模板文件例子
[aliases]
joe = /C=XZ/ST=Dessert/L=Snake City/O=Snake Oil, Ltd./OU=Research Institute/CN=Joe Average
# 用户组，方便批量授权
[groups]
admin = admin1,admin2
user = user1,user2
# 单版本库时直接配置路径
# 仓库的根路径
[/]
# 赋权给用户组
# 'rw'读写，'r'读，''无权限
@admin = rw
# 赋权给用户
admin1 = rw
# 赋权给别名代表的对象
&joe = r
# 赋权给所有人
* = r
# 赋权给匿名用户
$anonymous = r
# 给子路径单独配置权限
[/child]
@user = rw
# 在父路径设置了某个用户允许读写的话，默认子路径也是允许读写
# 必须显式设置为不可读写
@admin =
# 多版本库时配置标签是仓库名加路径
[repo1:/]
@admin = rw
# 多版本库时配置标签是仓库名加路径
[repo2:/]
@user = rw
```

## passwd

用户配置比较简单

```bash
[users]
# 格式为 用户名=密码
user1 = user1
```

# 创建仓库和启动关闭

创建仓库命令是`svnadmin create 仓库名`。
启动svn服务的命令`svnserve -d -r 仓库根路径`。

## 单版本库创建

单版本库只需要调用创建命令创建一个仓库，然后按自己的需求改配置文件即可。
__例如下__

```bash
cd /svn
# 建仓库
svnadmin create repository
cd repository
# 改配置
vim conf/svnserve.conf
vim conf/authz
vim conf/passwd
# 启动服务
# svnserve -d -r /svn/repository
svnserve -d -r .
```

单版本库也可以管理多个项目：只需要在第一次创建完仓库并启动服务后，checkout，在checkout出来的文件夹中添加多个子文件夹后提交，再在authz为各个子文件夹单独配置权限即可。

## 多版本库创建

多版本库创建时，需要先建一个空文件夹
然后再在这个文件夹中创建多个仓库，以及`authz`，`passwd`配置文件，配置用户和用户权限，在每个仓库中配置`svnserve.conf`文件将`authz-db`和`password-db`设为新建的两个配置文件，最后再启动`svnserve`即可

__示例__
```bash
cd /svn
mkdir multirepo
cd multirepo
svnadmin create repo1
svnadmin create repo2
vim authz
vim passwd
vim repo1/conf/svnserve.conf
vim repo2/conf/svnserve.conf
# svnadmin -d -r /svn/multirepo
svnadmin -d -r .
```

## 关闭

目前了解到的只有查出id然后kill

# 补充

1. 修改authz，passwd这两个文件时可以不重启svnserve
2. 另外Linux subversion客户端保存的密码在/root/.subversion/auth，windows用totorisesvn的话，在鼠标右键svn-->setting-->savedata里