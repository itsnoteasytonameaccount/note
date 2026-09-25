---
title: 百夫长统计
tags: 
- linux
- php
- mysql
date: 2018/04/29
---

# 前言
前天看到的新鲜东西，统计网站访问量的，本着试一试玩一玩的心态，记录过程
[官方文档](https://cn.centcount.com/doc.php?doc=%E7%99%BE%E5%A4%AB%E9%95%BF%E7%BB%9F%E8%AE%A1&article=%E5%AE%89%E8%A3%85%E7%99%BE%E5%A4%AB%E9%95%BF%E7%BB%9F%E8%AE%A1)
<!--more-->

# 内容

## 准备工作

***以下内容在centos上测试成功***

百夫长统计需要php+mysql+redis+nginx+php-fpm环境
并且需要php装了mysqli,redis,mbstring,gd,curl等扩展

并且需要为百夫长统计准备一位mysql用户，该用户需要有创建数据库权限，对创建的数据库拥有所有权限。
否则安装时会出现代码运行错误,当然，你可以直接用root用户(不建议)

### mysql相关

安装数据库方法请自行百度

可能用到的命令有
```bash
# 创建用户
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
# 查看用户信息（一般只需要知道host,user,password)
# 其中password是通过mysql自带password函数加密后的
# host:%代表可远程，127.0.0.1本地
USE mysql;
SELECT host,user,password FROM user;
# 显示用户所有权限
SHOW GRTANS FOR username@host;
# 分配用户权限（权限有all,create,select...其余不记得）
GRANT privileges ON database_name.table TO user@host;
# 收回用户权限
REVOKE privileges ON ON database_name.table FROM user@host;
# 收回所有权限(此处可能不对，记不清了)
REVOKE ALL PRIVILEGES,GRANT OPTION FROM user@host;
# 权限修改立即生效
flush privileges;
```

百夫长统计对mysql字符集也有要求，需要通过配置文件修改
mysql的配置文件是my.cnf
位置可能不确定，mysql help输出信息中有一段显示默认配置文件的位置
若实在找不到，通过以下命令全局搜索
```bash
find / -name my.cnf
```
配置文件修改完成后需要重启服务，有如下方法
```bash
# 找到pid
netstat -napt|grep mysqld
# 平滑重启
kill -USR2 pid
# 或者
service mysqld restart
# 或者
systemctl restart mysqld.service
```

### redis安装

cnetos安装redis只需要用以下指令

```bash
# 不加-y,以便确认是否选择了对的安装包
# yum install -y packagename表示不确认安装包，总是yes
yum install redis-server*
```

redis常用操作如下

```bash
# 后面可以没有配置文件路径，此时采用默认配置，不会后台运行
redis-server start /path/of/config
# 一般安装完redis,会有样板配置文件，通过find查找
find / -name redis.conf

# vim中搜索daemonize(似乎是这样拼)
# 将redis.conf文件中的daemonize no改为daemonize yes,此时再载入配置文件启动，就是后台运行了
# 当然也可以直接用参数进行后台启动
vim redis
redis-server start /path/of/config

# 连接数据库
# 默认hostname是localhost,port是6379
redis-cli -h hostname -p port

# 注意redis数据库操作不需要加';'
# 清库
flushall

# 查看key统计
info keyspace

# 选中index为1的database
select 1

# 显示当前库所有key
keys *

# 显示key的数据类型
type keyname

# 其余操作自行百度
```
### php配置


以下方法适用于centos上配置 php7扩展

0.  网上搜索都说可以直接通过yum install php-扩展名*
    一键安装，但我试过了没有用，在尝试下面方法前，先试试yum安装，以下属无奈之举

1.  添加mysqli扩展

    mysqli扩展我是通过重新编译安装添加的
    编译前你需要确认已安装了mysql-devel
    没安装的用以下命令安装

    ```bash
    yum install mysql-devel*
    ```
    安装后会多出一个mysql_config,用find查找

    编译安装php方法如下
    ```bash
    # 如果正在运行php-fpm,先kill
    ./configure --enable-fpm --with-mysqli=/path/of/mysql_config --disable-fileinfo


    # 1G以下内存不附带--disable-fileinfo参数会报错
    # 如果make出错，可能是缺少某些库支持
    make && make install


    # 安装完毕后需要修改配置文件
    # 可以自定义一个配置文件，例如
    vim php.ini
    # 在php.ini中加一行extension=mysqli.so
    php-fpm -c php.ini


    # 大功告成，在浏览器postinfo界面ctrl+f搜索mysqli,应该会有一块独立的详细信息

    # 如果还是没有mysqli，推荐以下方法检查
    # 在浏览器postinfo界面ctrl+f搜索extension_dir
    # 系统转到extension_dir路径下查看是否有mysqli.so
    # 或
    php -m
    # 查看是否加载了mysqli
    # 若有，可能是php-fpm的问题
    ```
2.  添加gd扩展
    如果你是用源码安装的php，
    转到php源码目录，cd至ext目录，里面有你需要的一切扩展的源码，
    例如安装gd
    ```bash
    # 先安装支持，缺一不可
    yum install freetype*
    yum install libjpeg*
    yum install libpng*

    cd php源码目录/ext/gd


    # 创建configure
    # phpize一般是安装php时自动添加在/usr/local/bin里的
    phpize
    # 如果提示phpize not found
    find / -name phpize


    # 再执行
    path/of/phpize


    # 接着如下,php-config的路径自行查找
    # --with-freetype-dir,--with-jpeg-dir可以不加等号及路径采用系统默认,如果提示缺少文件，再加也不迟
    # --with-freetype-dir,--with-jpeg-dir路径值请自行查找
    ./configure --with-freetype-dir=path  --with-jpeg-dir=path   --with-php-config=/usr/local/php/bin/php-config


    # 如果make出错，一般是需要某些支持库，自行上网百度
    make && make install


    # 安装完后，同样需要修改配置文件，
    # 添加一行extension=gd.so,并重启php-fpm
    ```
3.  添加curl扩展
    ```bash
    # 需要安装支持
    yum install libcurl*

    
    # config配置
    ./configure --with-php-config=path --with-curl-path=path
    # 其余同gd
    ```
4.  其余mdstring,redis扩展同上述方法

### 安装百夫长

其他配置都可以看官方文档

全部完成后进入安装界面
此处可能会有一个无法显示css等静态文件被403 forbidden的错误，原因是php-fpm默认只处理.php文件，
需要在php-fpm配置文件php-fpm.conf中或者其他被包含的配置文件中添加以下配置
```bash
security.limit_extensions=.php .ico .css .png .js .jpg
```
重启php-fpm

还有可能出现一个样式无法显示错误
原因是静态文件都被php-fpm当text/plain处理了，response里返回的时text/plain不被认为是css文件

其余按照官网步骤即可

[此处是官方示例网站](https://cn.centcount.com/manager.php?id=1610190000002&siteid=1610190000002001)
