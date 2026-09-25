---
title: centos各环境安装
date: 2020/03/28
tags:
- linux
- centos
- nginx
---
# 前言
装虚拟机，租服务器，帮同学朋友弄东西,每次都要重复一些步骤，每次都觉得有经验了，但每次都一堆问题，令人沮丧，总结记一记或许能好些
[这里有一份腾讯云的文档](https://cloud.tencent.com/document/product/213/38056)
<!-- more -->
# mysql
本文中安装的是mysql8
centos默认装了mariadb，但用习惯了mysql，并且水平差只会用Mysql
上mysql官网找到MySQL官方的 Yum Repository地址,rpm包，用wget下载并安装

```bash
# 下对应centos版本的仓库，否则会出问题
# 可以用uname -a查看自己centos版本
wget https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
yum install mysql80-community-release-el7-1.noarch.rpm #或rpm -i mysql80-community-release-el8-1.noarch.rpm
```
或直接
```bash
# 可选加-v输出细节，-h安装时打印文件的hash值
rpm -i https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
```
接着
```bash
yum install mysql-community-server
```
安装后会生成一个默认的配置文件
```bash
# 可以用如下命令找出
find / -name my.cnf
# 也可以用如下命令查看配置文件的搜索路径
mysql --verbose --help
```
配置文件中给了[官方配置建议的网址](http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html)
mysql8和mysql5.7默认的用户密码加密方式有点区别,可以通过配置设置
```bash
[mysqld]
default-authentication-plugin = mysql_native_password
# 默认拒绝用root用户启动，在此指定用户
user = mysql
# 设置报错输出路径,不设置时默认路径如下
log-error = /var/log/mysqld.log
# 跳过验证，忘记密码时使用
# skip-grant-tables=true
```
不存在mysql用户时用useradd创建用户，groupadd创建组
[useradd与adduser的区别](https://www.cnblogs.com/whitehorse/p/5847278.html)
```bash
# 用户是否存在查看
cat /etc/passwd|grep mysql
# 组是否存在
cat /etc/group|grep mysql
```
然后执行如下命令，下述内容可能在不同版本不同平台有些许不同
```bash
# 生成数据库文件
# 此步之后在log-error路径的输出中存在初始密码
mysqld --initialize
# 可在查看log文件路径
# 查看密码
cat /var/log/mysqld.log|grep password
# 找不到mysql.log文件的位置可以尝试用find找找
find / -name mysqld.log
# 如果实在找不到密码，在启动mysqld之前先跳过验证
# 在配置文件中修改，或者在启动mysqld时附加参数--skip-grant-tables
# window需要多一步安装服务,linux上不需要此步
mysqld --install
# 生成的文件位置可用如下命令查看
# 数据文件
mysqld --verbose --help|grep datadir
# pid文件
mysqld --verbose --help|grep pid-file
# 可以使用ll命令查看文件权限，所有者，所属组等
ll datadir
# 因为设置了user=mysql,需要修改文件夹所有者,-R为递归
# 不设置的话无法读写这些文件夹，导致无法启动
# datador默认为
chown mysqluser datadir -R
chgrp mysqluser datadir -R
chown mysqluser pid-file-dir -R
chgrp mysqluser pid-file-dir -R
# 启动服务
sevice mysqld start
# or
systemctl start mysqld
# or后台启动，这样启动的只能用kill来终止
mysqld &
# 跳过验证启动方法
mysqld --skip-grant-tables &
# 登录,初始密码有特殊字符需要用单引号引起来
mysql -uroot -p'yourpswd'
# 第一次登陆需要改初始密码
alter user root@'localhost' identified by 'yourpswd';
```
大功告成！

## 可能出现的错误

1.  安装mysql-community-server提示requires glibc2.28
    原因是安装错了仓库版本，但我一开始认为是glibc版本过低，尝试编译安装glibc,后来发现有些即使存在版本不低的lib也会报错才知道不是版本原因
2.  systemctl启动mysqld失败
    ```bash
    # 查看报错
    journalctl -xe
    # 报错pam unable to dlopen:/lib64/libnsl.so.1: symbol __nss_hash, version GLIBC_PRIVATE not defined in file libc.so.6 with link time reference
    #  /lib64/libc.so.6: version `GLIBC_2.25' not found (required by /lib64/libcrypt.so.1)
    # 但是通过如下命令检查这些定义都存在
    strings /lib64/libc.so.6|grep glibc
    strings /lib64/libc.so.6|grep __nss_hash
    # 解决办法，重启，因为GLIBC_2.25是刚安装的，得重启crond
    # centos 7以前的版本重启
    /etc/init.d/crond restart
    # centos 7重启
    systemctl restart crond
    # 但是仍可能存在其他问题，一般都是由于不用root启动带来的权限问题
    ```
    把mysqld.service文件里的用户改成root可以解决但可能会带来安全问题
    ```bash
    [Service]
    User=root
    Group=root
    ```
    直接注释掉也行
    ```bash
    [Service]
    # nginx的nginx.service配置文件里不存在User这两句
    # User=mysql
    # Group=mysql
    ```
## 自行编译安装glibc（没成功，仅记录安装过程学到的经验）
[在这个网站找合适版本](http://ftp.gnu.org/gnu/)的压缩包，编译安装也是个必要掌握的手段，曾经刚开始用centos时由于不知道yum依赖python，看到系统装的是python2.7，就把它卸了想装python3结果才发现yum不能用了,想尽办法重新安装yum，更新rpm，花了较长时间最后发现编译安装个python就能解决

1.  编译glibc的时候可能会提示某些组件版本过低,configure失败
    仔细看configure输出会把版本低或缺失的依赖列出来，或者可以对照glibc解压路径下的INCLUDE文件查看依赖那些组件，及版本号，[缺什么在这个网站找相应源码编译安装](http://ftp.gnu.org/gnu/)
2.  compiler版本低
    gcc版本低，编译安装高版本
3.  gcc自行编译安装，configure时提示缺少依赖
    先通过在源文件目录运行./contrib/download_prerequisites下载依赖包，需要较长时间，再逐个编译安装
4.  gcc编译时提示不存在32位编译环境
    按提示重新运行configure并且附带参数--disable-mutilib,编译时间较长
5.  gcc编译时提示缺乏必要的库
    ```bash
    #安装
     yum install glibc-headers
     yum install gcc-c++
     ```
### gcc编译完成后
```bash
#可以保存以前的gcc
mv /usr/bin/gcc /usr/bin/gcc-版本号
mv /usr/bin/g++ /usr/bin/g++-版本号
mv /usr/bin/c++ /usr/bin/c++-版本号
#创建软链接
ln -s /usr/local/bin/gcc /usr/bin/gcc
ln -s /usr/local/bin/g++ /usr/bin/g++
ln -s /usr/local/bin/c++ /usr/bin/c++
# 基本上命令行工具都可以通过--version,-v查看版本
# 都可以通过--help,-h,-?查看使用方法和参数列表
gcc --version
```
### 编译glibc
注意不要放在默认的/usr/local里否则会有提示，用--prefix设置到/usr里
或者可以安装到其他目录，然后通过环境变量来指定glibc
```bash
export LD_PRELOAD=/opt/glibc/libc-2.31.so
export LD_LIBRARY_PATH=/opt/glibc/lib
```
### 问题
一时手贱把libc.so.6删了想重新链接到新版本，才知道删了会导致系统指令不可用
解决方法
```bash
# 命令前设置LD_PRELOAD的值指定到哪加载共享对象
LD_PRELOAD=/usr/lib64/libc-2.31.so ln -s /usr/lib64/libc-2.31.so /usr/lib64/libc.so.6
```
### rpm方式安装glibc
编译glibc2.31成功,安装了glibc2.31后还是无法安装mysql,但编译glibc2.29和2.28总是失败
总是报slotinfo数组越界的问题，查看代码改了这处又有另一处
实在没办法换一种方式，用rpm安装
[glibc rpm包地址](http://rpmfind.net/linux/rpm2html/search.php?query=glibc-common&submit=Search+...&system=&arch=)
需要下glibc,glibc-common,glibc-all-langpacks三个包
```bash
# 需要同时安装，否则会报错缺少依赖
# 时间较长可添加-v选项查看输出
rpm -i glibc glibc-common glibc-all-langpacks
# 有时可加--force --nodeps选项，代表强制安装和不校验依赖
```
第一次rpm安装时没动静，提示.rpm: Header V3 RSA/SHA256 Signature, key ID 8483c65d: NOKEY
先关闭进程，处理步骤
```bash
# ctrl+z挂起后
jobs -l #查看挂起进程pid
kill -9 pid
```
[上述警告的原因和解决方案](https://www.cnblogs.com/doudou130702/p/5961233.html)
> 故障原因：
> 由于系统中先安装了你要安装的RPM包，而后你又安装了更低版本的，导致系统中的/lib64/libfreebl3.chk和/lib64/libfreebl3.so两个库文件被降级，与系统中其它的库文件版本不一致，从而导致rpm命令无法正常执行
> 此文给的解决办法是从别的机器上传过来

再次安装发现直接报错，BDB0087 DB_RUNRECOVERY: Fatal error, run database recovery
解决办法把/var/lib/rpm下的__db文件全删了

# jdk

[jdk下载，有rpm和可执行文件压缩包](https://www.oracle.com/java/technologies/javase-jdk14-downloads.html)
在链接上右键点击复制链接地址没用，wget下载rpm文件时被重定向到了另一个地址，下了个5kb的网页，当时还惊喜100多兆的文件瞬间下好，看了一下只有5kb。
需要在本地点下载获取复制到下载链接后取消下载，再用wget在服务器上下载，不过本地下载再用winscp上传快一些。

## 压缩包安装
下载压缩包解压到到自定文件夹后配置环境变量
> ```bash
  vim /etc/profile
  ```
[环境变量说明](https://www.cnblogs.com/shizhijie/p/9050029.html)
> profile中加入配置如下
  ```bash
  # 你的jdk文件路径
  export JAVA_HOME=/path/jdk/installed
  # java 1.5以后不需要配置CLASS_PATH也行
  export CLASSPATH=.:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib
  # 可执行文件路径
  export PATH=$JAVA_HOME/bin:$PATH
  ```
> 执行以下命令，读取环境变量。
  ```bash
  source /etc/profile
  ```
## tomcat
十几兆，下载比较慢本地下载后上传
[这里选择版本](https://archive.apache.org/dist/tomcat/)
也可以做个链接
```bash
cd /path/tomcat/installed
# 不做startup.sh文件的链接是因为这个单词太常见了，说不就和什么撞了
# 况且我也习惯用catalina启动,$PWD为当前路径
ln -s $PWD/bin/catalina.sh /usr/bin/catalina
# 然后可以执行看环境是否配好,以及使用说明
catalina
```
[关于startup.sh和catalina.sh的区别看这里](https://blog.csdn.net/yuhuafeixian/article/details/72792183?utm_source=blogxgwz4)不知道可不可信最好自己cat输出一下看看
关于各路径的官方说明,[详细的官方文档](https://tomcat.apache.org/tomcat-9.0-doc/introduction.html)
> /bin - Startup, shutdown, and other scripts. The *.sh files (for Unix systems) are functional duplicates of the *.bat files (for Windows systems). Since the Win32 command-line lacks certain functionality, there are some additional files in here.
> /conf - Configuration files and related DTDs. The most important file in here is server.xml. It is the main configuration file for the container.
> /logs - Log files are here by default.
> /webapps - This is where your webapps go.

几个重要的配置文件
```bash
# context.xml app部署配置
# 配置允许连接的Ip，域名啥的
$CATALINA_BASE/conf/[enginename]/[hostname]/[webappname].xml
$CATALINA_BASE/webapps/[webappname]/META-INF/context.xml
# Configuring Manager Application Access(设置管理程序的连接权限)
$CATALINA_BASE/conf/tomcat-users.xml
# 用户数据库实例，服务的端口，webapp位置，域名，上面tomcat-users.xml的位置等在server.xml里配置
$CATALINA_BASE/conf/server.xml
# 作用和app的contex.xml差不多，不过是全局所有web app生效的
$CATALINA_BASE/conf/context.xml
# 其余配置我也没太细去了解，具体看官网
```
[manager连接说明](https://tomcat.apache.org/tomcat-9.0-doc/manager-howto.html)
[配置说明](https://tomcat.apache.org/tomcat-9.0-doc/config/)
tomcat-users.xml如何配置在文件中有样例
用户角色说明
> manager-gui — Access to the HTML interface.
> manager-status — Access to the "Server Status" page only.
> manager-script — Access to the tools-friendly plain text interface that is described in this document, and to the "Server Status" page.
> manager-jmx — Access to JMX proxy interface and to the "Server Status" page.

### 禁止某个webapp外网访问
[这个](https://blog.csdn.net/laokaizzz/article/details/78423216)
在上文提到的app的context.xml中添加如下
```xml
<Context privileged="true" antiResourceLocking="false" docBase="${catalina.home}/webapps/manager">
	<Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="127\.0\.0\.1" />
</Context>
```
### 也可以和nginx配合
仅将catalina绑定到localhost:8080或者说127.0.0.1:8080然后nginx设置反向代理.
发往nginx的请求中:静态文件由Nginx处理动态由tomcat处理.
server.xml中配置如下
```xml
<!-- 在connector标签上加一个属性address -->
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000" address="127.0.0.1"
               redirectPort="8443" />...
               </Connnector>
```
nginx配置反向代理的方法见下文

# nginx
## 安装
可以编译安装，因为之前编译安装过，此次采用yum安装
[摘自腾讯文档](https://cloud.tencent.com/document/product/213/38056)
> 1.  执行以下命令，在 /etc/yum.repos.d/ 下创建 nginx.repo 文件。
  ```bash
  vi /etc/yum.repos.d/nginx.repo
  ```
> 2.  配置内容
  ```bash
  [nginx] 
  name = nginx repo 
  baseurl = https://nginx.org/packages/mainline/centos/7/$basearch/ 
  gpgcheck = 0 
  enabled = 1
  ```
> 3.  yum安装
  ```bash
  yum install -y nginx
  ```
配置路径在/etc/nginx/nginx.conf

```bash
# 链接
ln -s /path/nginx /usr/local/bin/nginx
# 测试配置文件
nginx -t
# 启动
service nginx start
# or
systemctl start nginx
# or
nginx
```
## 配置

初始配置文件内容如下
```bash
user nginx;
# 所属用户
worker_processes auto;
error_log /var/log/nginx/error.log;
# 错误输出位置
pid /run/nginx.pid;
# 运行中程序pid位置，用以reload,stop

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    # 服务实例
    server{
      # https协议在443端口
    	listen        80;#ip 是否启用ssl http版本等
    	# tcp端口和ip
    	server_name   ;# 域名
    	root          ;# 服务根路径
    	include       /etc/nginx/default.d/*.conf;
      # https需要启用ssl加密
      # ssl on;
      # 证书位置，需要自行申请腾讯云，阿里云都可以
      # ssl_certificate ******.crt;
      # 证书密匙
      # ssl_certificate_key ******.key;
      # ssl_session_timeout 5m;
      # ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
      # ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
      # ssl_prefer_server_ciphers on;
    	# 其他配置
    	location / {
    		# 设置根路径的默认文件
    		index index.html;
    	}
    	# 发生错误时，跳转页面，错误用错误码表示，这里是未找到
    	error_page 404 /404.html;
    }
}
```
大多数的配置都可以保持默认。
重点在于server中的内容
server决定的就是你的服务内容，其中几个参数上面注释中已有
在server中location配置可以针对某些特定的路径做处理
location中可以配置的东西有很多
下面一一介绍
1.  配置参数
    location一般格式如下
    location [匹配方法] 匹配表达式 {

    }
    其中匹配方法用符号选择也可不填
    ~代表不区分大小写的正则表达式
    =表示匹配某个精确的路径
    ~*表示不区分大小写的正则表达式

    匹配表达式则是用于  根据匹配方法匹配用户请求的路径名
    例如
    ```bash
    location ~ ^/py/* {
    	# nginx可以配置某一特定请求路径对应的位置
    	root   /path;
    	# 一些处理
    }
    ```
    下面有一种不用正则表达式的方式
    ```bash
    location /server/{
        proxy_pass localhost:8080/;
        # uwsgi配置
        # include uwsgi_params;
        # uwsgi_pass 127.0.0.1:1234;
    }
    # 此时对http://ip/server/a的访问会被转到localhost:8080/a
    ```
    nginx就会对**http://你的域名或ip/py/任意路径**路径下的请求进行处理

2.  静态文件
    据网上说，nginx擅长静态文件的处理，也就是图片，css样式，html文档，js脚本之类的文件
    举例location配置方法如下
    ```bash
    location ~ .(jpg|png|jpeg|ico)$ {
        expires 30d;
    }
    # 其中，~表示区分大小写的正则匹配，后面就可以用正则表达式来匹配需要处理的路径名了
    # expires定义文件过期的日期，静态文件一边不会做更改，因而可以设置较长的过期日期
    ```
3.  rewrite
    rewrite用于对某些路径重定向
    可以用于条件语句中（nginx配置中是可以有if条件语句的）,location中,和直接在server里
    我对rewrite是不甚了解的，只能说是用过，因而这里有个[详细的介绍页面](https://www.cnblogs.com/czlun/articles/7010604.html),当然，地址可能失效，因而同样记录在这篇博客里。
    以下内容搬自前面给出的博客里。
    >rewrite使用格式
     ```bash
     rewrite    <正则表达式>    <replacement>    [flag]
     ```
    >其中replacement为要rewrite到的内容，flag标记说明如下：
    >last  #本条规则匹配完成后，继续向下匹配新的location URI规则
    >break  #本条规则匹配完成即终止，不再匹配后面的任何规则
    >redirect  #返回302临时重定向，浏览器地址会显示跳转后的URL地址
    >permanent  #返回301永久重定向，浏览器地址栏会显示跳转后的URL地址

4.  反向代理
    location中，可以通过proxy_pass设置反向代理
    上面提到的nginx的优势是静态文件，动态服务就不是其专长了，因而可以对nginx将某些请求转发到特定的其他服务里
    格式
    ```bash
    proxy_pass ip或域名
    # 例
    # 注由于规则是正则表达式，因此星号的意义不能像许多其他地方一样作为通配符，正则表达式中*代表匹配前一项0到多次
    location ~ ^/server/ {
        proxy_pass http://127.0.0.1:8080;
        # 此处用户访问  http://你的域名/server/app  时
        # 实际上访问了  http://127.0.0.1:8080/server/app
        # 注不可在ip或域名后加上反斜杠 /
        # 否则location不得用正则表达式匹配,下面有介绍
    }
    location /server/ {
        proxy_pass http://127.0.0.1:8080/;
        # 此处用户访问  http://你的域名/server/app  时
        # 实际上访问了  http://127.0.0.1:8080/app
    }
    ```
    proxy_pass有一些配置用来优化服务器，以下提几个重要的
    ```bash
    client_max_body_size ;    # 允许客户端请求的最大单文件字节数
    proxy_buffer_size ;             # 设置代理服务器（nginx）保存用户头信息的缓冲区大小
    proxy_buffers ;               # 缓冲区，网页平均在32k以下的话，这样设置
    proxy_busy_buffers_size ;    # 高负荷下缓冲大小（proxy_buffers*2）
    # 其实，作为一个个人网站来说，不需要太过这些优化配置参数，因为这样一个
    # 小破站能同时有几个人访问就很不错了，所以不会对服务器带来什么负担，
    # 所以优化问题不是重点，可以待日后在学
    ```
## uwsgi参数
[uwsgi参数](https://blog.csdn.net/weixin_33802505/article/details/92478331)

## 重启

每次修改完配置后都需要重启，一般
```bash
nginx -s reload
```
当然修改完配置后可以测试配置是否正确。
```bash
nginx -t
```
重启失败一般有可能是配置格式不对，或Pid文件不存在，前者自行寻找原因,
若是pid文件不存在，创建一个后缀为pid的空文件，在配置文件里将Pid文件路径改为新建的文件路径，之后用以下方法强制重启
```bash
netstat -apt|grep nginx
# 执行完上面那步后你可以看到nginx Master进程的pid
kill pid
# 终结对应pid
nginx
# 再次启动
```
# php
腾讯云文档给出的方法

```bash
# 安装软件源
rpm -Uvh https://mirrors.cloud.tencent.com/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
# 安装软件
yum -y install mod_php72w.x86_64 php72w-cli.x86_64 php72w-common.x86_64 php72w-mysqlnd php72w-fpm.x86_64
# 启动php-fpm
systemctl start php-fpm
# 设为开机启动
systemctl enable php-fpm
```
[编译安装](https://www.cnblogs.com/rxbook/p/9106513.html)
查找配置文件
```bash
# php-fpm
php-fpm -t
# php
find / -name php.*
```
## 扩展
# 多版本python
系统默认装的2.7,从python官网下载python3源码，编译安装
编译前需要准备必要的依赖否则报错部分可选模块编译失败
```bash
# 可能需要安装,未注释的为不确定是否有效
yum install ncurses-devel #_ncurses
yum install libuuid-devel
yum install -y zlib zlib-devel
yum install tkinter 
yum install tcl-devel tk-devel 
yum install lzma
yum install libgdbm-dev
yum install xz-devel
yum install readline-devel* #readline
yum install uuid-devel* #_uuid
yum install sqlite-devel* #_sqlite3
yum install gdbm-devel* #_dbm _gdbm
yum install bzip2* #_bz2
# 其他的缺什么通过yum搜一下
# 例如提示_sqlite3缺乏依赖，就执行如下
yum search sqlite
# yum搜不到再百度
# 问题都解决后make install搞定
```
python 2.7装在/usr里
python3安装时未设置prefix所以装在/usr/local
需要建立链接到/usr/bin
```bash
ln -s /usr/local/bin/python3.8 /usr/bin/python3
```
## 可能的错误
### ncurses error: implicit declaration of function ‘setupterm’; did you mean ‘set_ter’
具体原因不明，但做了如下尝试，成功解决
make clean
重新make
还不行可能得重新configure
### INFO: Could not locate ffi libs and/or headers
```bash
yum search ffi
# 找到了libffi-devel
yum install libffi-devel
```
### INFO: Can't locate Tcl/Tk libs and/or headers
编译安装tcl/tk,[源码下载](http://www.tcl.tk/software/tcltk/download.html)

编译tk时报错fatal error: X11/Xlib.h: No such file or directory
```bash
yum install libXt-devel
```
### uuid_create参数过多
uuid_create声明在uuid.h
根据查看Modules/_uuidmodule.c发现调用uuid_create传入了两个参数
/usr/inlcude/uuid.h里声明的uuid_create只有一个参数，可能是库的版本问题
因为重复安装了一些东西，发现/usr/include/uuid/uuid.h文件也存在,但其中似乎没有声明uuid_create，不过声明了uuid_generate_time_safe
改Modules/_uuidmodule.c，在开头加上
```c
// uuid/uuid.h存在，编译时引入/usr/include/uuid/uuid.h不引入/usr/inlcude/uuid.h
#define HAVE_UUID_UUID_H
// uuid_generate_time_safe存在，编译时执行uuid_generate_time_safe不执行uuid_create
#define HAVE_UUID_GNERATE_TIME_SAFE
// 重新make clean&&make成功
// 有时候还出错可能需要重新configure且重新make
// 最后
make install
```
### Could not build the ssl module!
Python requires an OpenSSL 1.0.2 or 1.1 compatible libssl with X509_VERIFY_PARAM_set1_host().
LibreSSL 2.6.4 and earlier do not provide the necessary APIs, https://github.com/libressl-portable/portable/issues/381
意思是：Python需要具有X509_VERIFY_PARAM_set1_host（）的OpenSSL 1.0.2或1.1兼容libssl。
LibreSSL 2.6.4和更早版本不提供必要的API，https：//github.com/libressl-portable/portable/issues/381
```bash
# 搜索需要的包
yum search openssl
# 但我把所有包都装了也没用
yum install openssl*
# 只好下源文件自行编译，编译过程较顺利
```
[openssl源码下载](https://www.openssl.org/source/mirror.html)

## pip安装
首先用whereis pip看看有没有，有就链接到/usr/bin,没有就装
> 一、脚本安装pip
> 首先下载get-pip.py脚本，然后运行即可安装：
  ```bash
  $  curl -O https://bootstrap.pypa.io/get-pip.py #或wget
  $  python get-pip.py
  ```
> 二、使用包管理软件安装
  1、首先安装epel扩展源：
  ```bash
  $ sudo yum -y install epel-release
  ```
> 2、然后安装python-pip
  ```bash
  $ sudo yum -y install python-pip
  ```
之后就可以用pip安装包了
```bash
# virtualenv
pip install virtualenv
# python3的pip链接成pip3
ln -s /dir/install/pip /usr/bin/pip3
# 就可以用pip3管理python3的包了
# flask
pip3 install flask
# 也可以这样装
python3 -m pip install uwsgi
```
pip配置国内源
阿里云:[http://mirrors.aliyun.com/pypi/simple/](http://mirrors.aliyun.com/pypi/simple/)
中国科技大学:[https://pypi.mirrors.ustc.edu.cn/simple/](https://pypi.mirrors.ustc.edu.cn/simple/)
豆瓣:[https://pypi.douban.com/simple/](https://pypi.douban.com/simple/)
清华大学:[https://pypi.tuna.tsinghua.edu.cn/simple/](https://pypi.tuna.tsinghua.edu.cn/simple/)
```bash
# 较早版本的pip似乎没有这条命令
pip config set global.index-url=yourhost
# 执行完命令后会提示配置写入了/root/.config/pip/pip.conf
```
[centos上配置也可以看这里（未验证是否有效）](https://www.cnblogs.com/kazihuo/p/11051319.html)

### setuptools

除了pip很多库可以在源码中用`python setup.py install`安装，一般这种安装方法需要setuptools,[这里](http://pypi.python.org/pypi/setuptools )下载源码后，通过`pythoon setup.py install`安装后，其他库就可通过同样的命令安装了


## uwsgi配置
```ini
[uwsgi]
#用户和组
uid = nginx
gid = nginx
# 设置nginx uwsgi_pass时用
socket = 127.0.0.1:8000
# 直接作为服务器时用
; http = 127.0.0.1:8000
# 状态服务
; stats = 127.0.0.1:8000
master = true
vhost = true
workers = 8
reload-mercy = 10
vacuum = true
max-requests = 10000
limit-as = 1024
buffer-size = 65535
# pid file重启或停止时用
pidfile = /projects/pid/uwsgi.pid
daemonize = /projects/log/uwsgi.log
# 项目网页路径
#chdir = /projects/python/html
callable = app
chmod-socket = 660
enable-threads = true
# 启动文件
wsgi-file = /projects/python/app.py
```
[uwsgi常用参数配置](https://www.cnblogs.com/ajianbeyourself/p/12823295.html)
使用

```bash
# 启动
uwsgi --ini /projects/config/uwsgi.ini
# 停止
uwsgi --stop /projects/pid/uwsgi.pid
# 重启
uwsgi --reload /projects/pid/uwsgi.pid
```

设置nginx代理到uwsgi时会出现502,查看`/var/log/nginx/error.log`下的报错，得到connect() to xxxxx failed (13: Permission denied) while connecting to upstream原因是selinux的权限设置，解决办法是如下命令

```bash
# 取消的话最后参数设成0就可以
setsebool -P httpd_can_network_connect 1
# 也可以用如下命令，未验证，似乎需要重启，反正我没重启不生效
setenforce 0
# 也可以直接改/etc/selinux/config需要重启
```

# nodejs
[阿里云镜像](https://npm.taobao.org/mirrors/node/)
同样有编译好的可执行文件，注意看包名，分平台的是可执行文件，不分平台是源码
解压后放到自己喜欢的位置，然后把其中bin文件夹里的三个可执行文件位置连接到/usr/bin或者设置环境变量,最好是加全局变量，之后全局安装的可执行文件都在Bin里
```bash
# 可以看到全局安装的包的位置
npm --help
# npm配置查看
npm config list
# npm所有默认配置
npm config ls -l
# 查看如何设置
npm help config
```
## pm2
nodejs的进程管理工具
```bash
npm i -g pm2
```
# maven
[镜像网站](https://mirrors.cnnic.cn/apache/maven/)
有编译好的二进制文件
也可以yum安装
```bash
# 可以搜到一个maven.noarch
yum list maven
# 尝试安装发现要110个依赖，有点担心是否装错
yum install maven
```
[网上搜索到用如下方法yum安装](https://www.cnblogs.com/sanduzxcvbnm/p/11550819.html)
>```bash
wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
yum -y install apache-maven
```
设置环境变量
```bash
export MAVEN_HOME=/usr/local/maven
export PATH=$PATH:$MAVEN_HOME/bin
```

# redis安装
centos yum安装redis如下
```bash
# 先列出一下
yum list redis
# 找到了确定是这个在安装
yum install redis*
```

redis常用操作如下

```bash
# 后面可以没有配置文件路径，此时采用默认配置，不会后台运行
redis-server /path/of/config
# 一般安装完redis,会有样板配置文件，通过find查找
find / -name redis.conf

# vim中搜索daemonize(似乎是这样拼)
# 将redis.conf文件中的daemonize no改为daemonize yes,此时再载入配置文件启动，就是后台运行了
# 当然也可以直接用参数进行后台启动
vim redis-config-path
redis-server /path/of/config

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
# svn安装
```bash
yum install subversion
```
svnadmin创建仓库
```bash
svnadmin create /your/repository/dir
```
配置文件
```bash
# authz中:
[aliases]
# joe = /C=XZ/ST=Dessert/L=Snake City/O=Snake Oil, Ltd./OU=Research Institute/CN=Joe Average
[groups]
userg = users #用户组，方便进行分组权限分配
[test:/] #test项目的根目录
@userg = rw
# users = rw
#[/] #根目录权限
#@userg = rw
#users = rw
# [repository:/baz/fuz]
# @harry_and_sally = rw
# * = r
# passwd中:

[users] #用户
# harry = harryssecret
# sally = sallyssecret
users = users

# svnserve.conf中:
[general]
### The anon-access and auth-access options control access to the
### repository for unauthenticated (a.k.a. anonymous) users and
### authenticated users, respectively.
### Valid values are "write", "read", and "none".
### Setting the value to "none" prohibits both reading and writing;
### "read" allows read-only access, and "write" allows complete
### read/write access to the repository.
### The sample settings below are the defaults and specify that anonymous
### users have read-only access to the repository, while authenticated
### users have read and write access to the repository.
anon-access = none
auth-access = write
password-db = passwd #密码配置文件
authz-db = authz #验证配置文件
realm = svntest #描述
```

另外Linux删除用户默认密码在/root/.subversion/auth
windows用totorisesvn在鼠标右键svn-->setting-->savedata里

# 对顽固问题可能有用的解决方案
在一个系统上编译不通过，去其他相同版本系统编译（或者不同系统交叉编译）说不定能行，曾有过云服务器编译失败，本地虚拟机编译成功复制过去还能用的经历