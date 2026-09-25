---
title: 彻底删除mysql服务
tags: mysql
date: 2018/10/20
---
# 解决方案

不知为何mysql服务花了很长时间才删除，记录办法。

[点击这里](http://www.cnblogs.com/fanbi/p/6432838.html)

__补充__
事实上使用`sc`命令似乎就足够了，在打开`services.msc`的情况下，使用`sc delete`命令并不会立刻删除服务的注册表，服务也仍然存在于服务列表中，因此就容易造成服务没被完全删除的假象，此时只要刷新一下servies.msc的窗口即可看到服务被完全删除，

# 其他问题
navicat连接不上数据库
可能是用户名密码加密方式问题，会影响你创建用户时存入数据库的用户密码，查看配置文件设置选项。或直接在创建用户时添加条件
例如
```bash
 ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '9527';
```

# group by选择语句问题
可能是sql_mode设置出了问题，[点击这里](https://www.cnblogs.com/fireporsche/p/8618691.html)查看sql_mode问题

# windows上启动mysql
可能的方法
```bash
net start service_name
# 此时用net stop停止
mysqld
# 此时直接taskkill
```
添加服务的方法
```bash
mysqld --install service_name
mysqld --remove service_name

# 还有一种用sc的方法
sc start exe_path
# 删除
sc delete service_name
# 此时只是禁用还需删除注册表
```