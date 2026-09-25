---
title: 服务器简单的nginx配置(1)
tags: 
- nginx
- linux
date: 2018/04/14
---

# 前言

nginx，比较常用的静态文件服务器，记录一下最近学到的用法
<!--more-->

## 安装

虽然找到一个比较完整的[现成的（点这里）](http://www.nginx.cn/install)安装教学，还是在此记录一下
[另一个现成的](https://www.cnblogs.com/sign-ptk/p/6723048.html)
安装前需要先安装支持库，prec,ssl,zlib.

1.下载
```bash
wget  http://nginx.org/download/nginx-1.6.2.tar.gz
```
2.解压
```bash
tar -xzvf nginx-1.6.2.tar.gz
```
3.安装
```bash
./configure
make
make install
```
这里可以在configure后面加编译选项，具体有什么参数也可以参看以上给出的地址里的，
当然什么都不加采取默认设置也没什么问题，此时安装完后所属用户默认为nginx(安装时自动添加),组默认为nginx(安装时自动添加)
默认路径似乎各有不同，可以用以下命令很快找出
```bash
whereis nginx
```

## 配置

若采取默认配置，配置文件应当是在/etc/nginx/nginx.conf

若该路径没有
```bash
find /etc -name nginx.conf
```
即可找到

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
    	listen        80 ip;
    	# tcp端口和ip
    	server_name   ;# 域名
    	root          ;# 服务根路径
    	include       /etc/nginx/default.d/*.conf;
    	# 其他配置
    	location / {
    		index index.html;
    		# 设置根路径的默认文件
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
    [不错的总结](https://www.cnblogs.com/ustc-anmin/p/10878636.html)
    location [匹配方法] 匹配表达式 {

    }
    其中匹配方法用符号选择也可不填
    ~代表区分大小写的正则表达式
    =表示匹配某个精确的路径
    ~*表示不区分大小写的正则表达式
    ^~匹配到当前规则后立刻运用，不向下搜索,不知道为何alias不使用这个，会无限重定向

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
    }
    # 此时对http://ip/server/a的访问会被转到localhost:8080/a
    ```
    nginx就会对**http://你的域名或ip/py/任意路径**路径下的请求进行处理

2.  静态文件

    据百度,nginx擅长静态文件的处理，也就是图片，css样式，html文档，js脚本之类的文件
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
    >```bash
    >rewrite    <正则表达式>    <replacement>    [flag]
    >```
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
    location ~ ^/server/ {
        proxy_pass http://127.0.0.1:8080;
        # 此处用户访问  http://你的域名/server/app  时
        # 实际上访问了  http://127.0.0.1:8080/server/app
        # 注不可在ip或域名后加上反斜杠 /
        # 否则location不得用正则表达式匹配,下面有介绍
    }
    # 注由于规则是正则表达式，因此星号的意义不能像许多其他地方一样作为通配符，正则表达式中*代表匹配前一项0到多次
    location /server/ {
        proxy_pass http://127.0.0.1:8080/;
        # 此处用户访问  http://你的域名/server/app  时
        # 实际上访问了  http://127.0.0.1:8080/app
    }
    ```
    proxy_pass有一些配置用来优化服务器，以下提几个重要的
    ```bash
    client_max_body_size ;   # 允许客户端请求的最大单文件字节数
    proxy_buffer_size ;             # 设置代理服务器（nginx）保存用户头信息的缓冲区大小
    proxy_buffers ;               # 缓冲区，网页平均在32k以下的话，这样设置
    proxy_busy_buffers_size ;    # 高负荷下缓冲大小（proxy_buffers*2）
    # 其实，作为一个个人网站来说，不需要太过这些优化配置参数，因为这样一个
    # 小破站能同时有几个人访问就很不错了，所以不会对服务器带来什么负担，
    # 所以优化问题不是重点，可以待日后在学
    ```
## 运行

控制台输入nginx就行了，如果提示unfound,就通过whereis找到名为nginx的可执行文件的位置，运行它就够了,当让要将其添加到环境变量。
```bash
ln -s /path/nginx /usr/local/bin/nginx
```

## 重启

每次修改完配置后都需要重启，一般
```bash
nginx -s reload
```
重启失败一般有可能是配置格式不对，或Pid文件不存在，前者自行寻找原因。
当然修改完配置后可以用
```bash
nginx -t
```
测试配置是否正确。
若是pid文件不存在，创建一个后缀为pid的空文件，在配置文件里将Pid文件路径改为新建的文件路径，之后用以下方法强制重启
```bash
netstat -apt|grep nginx
# 执行完上面那步后你可以看到nginx Master进程的pid
kill pid
# 终结对应pid
nginx
# 再次启动
```

## 结语
本页里引用的两个是对我帮助较大存了书签的，还有不少博客对我产生了帮助，忘了保存就不找出来了