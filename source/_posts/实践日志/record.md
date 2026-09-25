---
title: rainmeter皮肤google翻译工具编写
data: 2020/07/31
tags:
  - python
  - javascript
  - rainmeter
---

# 前言

由于写代码起变量名有时需要查翻译，但每次都要打开关闭一个 google 翻译页面非常不方便，因此想做一个一直处于屏幕顶部的随时可用的 google 翻译工具栏。

以下为经验记录

<!-- more -->

# 先说结论

效果如下

![图片无法显示](../image/result.gif)

# 本文中的内容概述

1. rainmeter 皮肤编写。
2. 看rainmeter文档时，由于加载比较慢，用 python 写代理程序转发请求用来加快文档的网页加载速度。
3. 写 flask 程序时，用代码实时监听文件变化执行特定命令。
4. gbk 转 unicode

## rainmeter 皮肤编写

用 rainmeter 的 webpaser 发送请求，lua 脚本处理 json、拼接字符串。

> 问题：本以为用 lua 脚本会很简单，可以发请求、处理一条龙，用了才知道 lua 本身不支持网络编程，其次 rainmeter 中使用 lua 时，rainmeter 文档明确指出无法使用需外部编译的第三方库，无法用 require 引入文件，只能用 dofile 引入.lua 文件，然后字符串编码还有问题。

## python 转发请求

看 rainmter 文档时由于网络问题导致文档加载要半天甚至无法加载。在服务器上用 wget 下载页面很快，因此用 flask 框架写了个 python web 服务端程序通过服务器转发请求。

## python 监听文件夹内文件及子文件夹文件变化，python 多进程，python 调用系统命令

写转发程序时没认真看 flask 的文档，不知有 debug 模式。写了个 watcher.py，用于监听当前目录及子目录下文件修改，重启进程。失败了（在完成前发现有 debug 模式），但算是学到点东西。

## google 翻译接口调用方法

从 google 翻译网站上用 network 找的接口

接口调用时可以传一个 tk 参数，传 tk 参数时，返回结果更详细（包括拼音，近义词），不传 tk 返回简略结果。
由于想要详细结果，以及不传参数 tk 时接口返回的中文字符在 rainmeter 中是乱码(不知原因，用 codepage 指定编码没用)，因此需要生成 tk。

## js GBK 乱码处理

rainmeter 用 gbk 格式保存字符，js 字符格式是 utf16，是 unicode 的一种实现，unicode 和 gbk 不兼容，所以 rainmeter 把 gbk 中文发送给 js 后，js 将其当成只有单字节字符的范围属于 0x00~0xff 的 iso-8859-1 编码处理，因此出现乱码。所以还得做编码转换。
gbk unicode 不兼容，转化只能查表，附 json 格式的 gb2312-->unicode 编码映射表。

# 详细

## 代理请求

请求发到服务器上的 python 程序，由 python 程序将请求转发到真正的地址，并且将响应转发回来

<details>
<summary>python代码</summary>

```python
# config.py

rootpath = "/"
url = "https://docs.rainmeter.net/"
# url = "https://www.baidu.com"
# 转发哪些header
headers_filter = ["User-Agent", "Content-Type", "Cookie", "Accept",
                  "Accept-Encoding", "Cache-Control", "Accept-Language", "Connection", "Host"]#, "Referer"]
# 不转发哪些header
headers_filter_exclude = []  # "Host"]
# 不响应哪些header
headers_exclude = ["Set-Cookie", "Content-Encoding"]
# 响应哪些header
headers_include = ["Server", "Content-Type", "Cache-Control", "Connection",
                   "Content-Encoding", "Content-Length", "Date", "Pragma", "Transfer-Encoding"]

# app.py

from werkzeug.serving import run_simple
from flask import Flask, request
from werkzeug.routing import BaseConverter
import math
import random
import re
from get_params import get_params
from web import get_response
from config import rootpath, headers_filter, headers_filter_exclude

# flask支持正则匹配路径的实现，copy from https://cloud.tencent.com/developer/article/1519919
class RegexConverter(BaseConverter):
    def __init__(self, url_map, *args):
        super(RegexConverter, self).__init__(url_map)
        self.url = url_map
        print(args, url_map)
        self.regex = args[0]

    def to_python(self, value):
        return value


app = Flask(__name__+str(random.randint(0, 1000)))

app.url_map.converters['re'] = RegexConverter

@app.route(rootpath+'<re(r".*"):path>')
def server(path):
    return get_response(path, params=request.args, method=request.method, data=request.data, headers=get_headers(request.headers.__str__()))


def get_headers(headers):
    headers = headers.split("\r\n")
    headers = [header.split(": ") for header in headers if len(
        header) > 0 and header in headers_filter] #and header not in headers_filter_exclude ]
    dic = {}
    for [name, value] in headers:
        dic[name] = value
    return dic

if __name__ == "__main__":
    # app.run("127.0.0.1", random.randint(8000, 9000))
    dic = get_params(("p:h:", True))
    app.run(dic["-h"], dic["-p"], debug=true)
    # start_server("127.0.0.1", 8100)

# get_params.py
# 工具函数get_params,获取特定的命令行参数

import sys
from getopt import getopt, GetoptError


def decode_argv(t):
    # 处理单个元组
    lens = len(t)
    if lens == 0:
        return "", []
    lens2 = len(t[0])
    if lens == 1:
        # len(str)==1
        # if lens2 == 1:
        if lens2 >= 1:
            return t[0], []
        # ()
        elif lens2 == 0:
            return "", []
        # len(str)>1
        # else:
        #     return "", [t[0]]
    elif lens == 2:
        # (str,True)
        if t[1]:
            return t[0], []
        # (str,False)
        else:
            return "", [t[0]]
    else:
        # (str,True,any)
        if t[1]:
            return t[0], []
        else:
            # (str,False,True)
            if t[2]:
                return "", [t[0]+"="]
            # (str,False,False)
            else:
                return "", [t[0]]
    pass


def decode_argv_list(li):
    # 处理多个元组
    def1 = ""
    def2 = []
    for item in li:
        # get_params((str,bool,bool),(str,bool,bool),...)
        if type(item) is list or type(item) is tuple:
            a1, a2 = decode_argv(item)
        # get_params(str,str,...)
        else:
            a1, a2 = decode_argv([item])
        def1 += a1
        def2 += a2
    return def1, def2


def get_params(*argvs):
    lens = len(argvs)
    def1 = ""
    def2 = []
    if lens == 0:
        return {}
    if type(argvs[0]) is tuple:
        # get_params((str,bool,bool),(str,bool,bool),(str,bool,bool),...)
        def1, def2 = decode_argv_list(argvs)
    elif type(argvs[0]) is dict:
        li = []
        for name in argvs[0].keys():
            # get_params({"arg1":(bool,bool),"arg2":(bool,bool),...})
            if type(argvs[0][name]) is tuple:
                li.append([name]+list(argvs[0][name]))
            # get_params({"arg1":bool,"arg2":bool,...})
            else:
                li.append([name, argvs[0][name]])
        def1, def2 = decode_argv_list(li)
    elif type(argvs[0]) is str:
        # get_params(str,str,...)
        if lens == 1 or type(argvs[1]) is str:
            def1, def2 = decode_argv_list(argvs)
        # get_params(str,bool,bool)
        else:
            def1, def2 = decode_argv(argvs)
    else:
        print("params type error!")
        return {}
    try:
        opts = getopt(sys.argv[1:], def1, def2)[0]
    except GetoptError:
        print("getopt error")
        return {}
    result = {}
    for opt, arg in opts:
        result[opt] = arg
    return result


if __name__ == "__main__":
    # get_params()
    # get_params("ova")
    # get_params("ova", True)
    # get_params("")
    # get_params((), ("ova"))
    # print(get_params({"ova": (True, True)}))
    pass


# web.py
# 发送请求

from config import url, headers_exclude, headers_include
from flask import Response
import requests
headers = {
    "User-Agent": "mozilla/5.0 (windows nt 10.0; win64; x64) applewebkit/537.36 (khtml, like gecko) chrome/84.0.4147.105 safari/537.36"
}
params = {
}


def get_response(path, method="GET", params=params, headers=headers, data=""):
    if method == "GET":
        return to_flask_response(requests.get(url+path, params=params, headers=headers))
    elif method == "POST":
        return to_flask_response(requests.post(url+path, params=params, headers=headers, data=data))


def to_flask_response(res):
    response = Response()
    response.status_code = res.status_code
    # print(res.headers, "reponse.header", response.headers)
    headers = ""
    for name in res.headers:
        if name in headers_exclude:
            continue
        # res.headers[name] = res.headers[name].replace("domain","do")
        # if name in headers_include:
        response.headers[name] = res.headers[name]
    response.data = res.content
    print(res.cookies.items())
    for name, val in res.cookies.items():
        response.set_cookie(name, val)
    # response.set_cookie(res.cookies)
    return response


if __name__ == "__main__":
    res = requests.get(url, params=params, headers=headers)
    print(res.text)
    pass
```

</details>

### 配置 uwsgi

```ini
; uwsgi.ini
[uwsgi]
#user and group
uid = nginx
gid = nginx
#直接部署
#http = 0.0.0.0:8000
#通过nginx设置uwsgi_pass部署
socket = 127.0.0.1:8000
stats = 127.0.0.1:8001
master = true
vhost = true
workers = 8
reload-mercy = 10
vacuum = true
max-requests = 10000
limit-as = 1024
#64kb
buffer-size = 65536
pidfile = /projects/pid/uwsgi.pid
daemonize = /projects/log/uwsgi.log
#网页文件路径
chdir = /projects/python
module = app
callable = app
chmod-socket = 660
enable-threads = true
#启动程序
uwsgi-file = /projects/python/app.py
```

```conf
# nginx.conf
location ~/ {
    uwsgi_pass host_name;
    include uwsgi_params;
}
```

### 遇到的问题

设置 http 地址方式部署 uwsgi 时，返回响应不过滤掉`Transfer: chunked`会导致响应解析失败，找了许久原因，用 postman 请求得到提示`Transfer: chunked`和`Content-Length:`冲突，不能同时存在。
但使用 nginx 反向代理到 uwsgi 时，不过滤`Transfer:chunked`也不会有问题。

曾经写过用 c 处理 http 请求动态生成响应报文的程序，由于无法计算出`Content-Length`,而不响应`Content-Length`,改加`Transfer:chunked`,因为据网上说这个可以代替`Content-Length`，同样导致报文解析时报错。
因此认为不是`Transfer: chunked`与`Content-Length:`冲突，而是`Transfer: chunked`这个头某些情况会导致报错，具体原因未知

请求转发过程中若是将请求头 headers 原封不动转发，会被检测到问题，响应 403 无权限
原因：服务器会检查 Referer，转发后 Referer 是不匹配的，把 Referer 过滤了或者伪造 Referer

## 监听文件变化并重启进程

思路：循环遍历当前文件夹，若文件有改变，将之前创建的子进程关闭，调用`python3 app.py -h 127.0.0.1 -p 端口号`命令来开启新的子进程

### 主要代码

```python

# watcher.py
import os
from sys import platform
import time
import json
import re
from cocurrent import ProcessCocurrent, start_process, co_exec
import subprocess
# file = open("./watcher.json")
# jsonTxt = file.read()

# 读配置
class Config:
    def __init__(self, config_json=None):
        if config_json == None:
            self.config_json = json.load(open("./watcher.json"))
            return
        self.config_json = config_json

    def is_exclude(self, path):
        exclude_list = self.config_json["exclude"]
        if exclude_list == None:
            return False
        for p in exclude_list:
            if not re.search(p, path) == None:
                return True
        return False

    def is_include(self, path):
        include_list = self.config_json["include"]
        if include_list == None:
            return True
        for p in include_list:
            if not re.search(p, path) == None:
                return True
        return False

    def get_path(self):
        return self.config_json["path"]

# 遍历文件
class FileScan:
    def __init__(self, config=None):
        if config:
            self.config = config
            return
        self.config = Config()

    def get_file_list(self, path):
        return os.listdir(path)

    def get_file_list_r(self, path):
        if not self.is_dir(path):
            if self.config.is_include(path):
                return [path]
            else:
                return []
        file_list = self.get_file_list(path)
        file_list = [
            path+"/"+p for p in file_list if not self.config.is_exclude(p)]
        result = []
        for p in file_list:
            result += self.get_file_list_r(p)
        return result

    def is_dir(self, path):
        return os.path.isdir(path)

    def is_updated(self, path):
        return

    def is_change(self, path):
        return

# 管理进程
class Watcher:
    def __init__(self, cmd):
        config = Config()
        self.dic = {}
        self.path = config.get_path()
        self.scanner = FileScan(config)
        self.p = None
        self.cmd = cmd

    def start(self):
        while True:
            # print("started")
            files = self.scanner.get_file_list_r(self.path)
            # print("watch", files)
            if self.is_changed(files):
                print("changed")
                if self.p:
                    print("kill p")
                    self.p.kill() and self.p.join()
                    time.sleep(1)
                self.p = ProcessCocurrent()
                self.p.execl(self.cmd)
                print("started"+str(self.p.pid))

    def is_changed(self, files):
        flag = False
        for path in files:
            state = os.stat(path)
            if path in self.dic.keys():
                flag |= state.st_mtime != self.dic[path]
            else:
                flag = True
            self.dic[path] = state.st_mtime
        return flag


if __name__ == "__main__":
    Watcher("cls &&python3 -B app.py -h 127.0.0.1 -p 8100").start()

```

### 多进程及执行命令

python 中多进程编程库找到有`multiprocessing`，以及`subprocess`
在 linux 下还可以用`os.fork`创建进程，用法以目前感觉来说和 c 的 fork 没什么区别
执行命令有`os.exec*`族,`os.system`,`subprocess.Popen`等等

#### 创建子进程

1. os.fork
   复制一个进程，子进程返回 0，父进程返回子进程 id，可在程序中以此区分父子进程
2. multiprocessing.Process
   调用方法两种，直接调用`Process(target=要执行函数, args=参数列表)`
   继承调用，例如

   ```python
   class ProcessCocurrent(Process):
     def __init__(self, callback=None, *args):
         self.callback = callback
         self.args = args
         super().__init__()

     def run(self):
         if self.cmd:
             cmdli = self.cmd.split(" ")
             os.execvp(cmdli[0], cmdli)
         if self.callback:
             self.callback(*self.args)

     def setCallback(self, callback, *args):
         self.callback = callback
         self.args = args

     def execl(self, cmd):
         self.cmd = cmd
         self.start()
         return self
   ```

其余用法可百度

#### 调用命令

1. [os.exec\*用法](https://www.cnblogs.com/johnyang/p/10888780.html)，`os.exec*`会覆盖当前进程，此句之后的语句不会再执行，windows 上`os.exec*`执行命令后进程 id 会改变，linux 上不会
2. `os.system`相当于在命令行下执行某个命令，执行时,创建新进程，阻塞当前进程
3. `subprocess.Popen`相当于打开一个指定的程序（在 windows 下和`os.system`有点区别，linux 上有没有区别不知），不阻塞当前进程，创建新进程。
   关于上述提的“命令行下执行某个命令”和“打开一个指定程序”的区别在于：
   1. 前者需要先创建 cmd，再通过 cmd 进程执行命令或打开某个可执行程序，打开可执行程序时时产生的进程作为该 cmd 进程的子进程存在，例如命令行里运行 tomcat 的 catalina，当命令行关闭时 catalina 也关了。
   2. 而靠双击打开 catalina 进程则是独立的一个进程。
4. `subprocess.Popen`的参数 shell 若设置为 True 时，windows 下相当于在命令前添加了 cmd /c，linux 下相当于添加了/bin/sh -c，即和`os.system`等效但不会阻塞。（下述情况仅在 windows 下测试存在，linux 未知）这会导致某些情况下返回进程对象的进程号和实际上你执行的命令产生的进程号不一致,获得的事实上是 cmd 程序的进程号。例如：

   ```python
   p = subprocess.Popen('python3 test2.py',shell = True)
   ```

   上述语句其实会创建两个进程，一个 cmd 进程，一个由 cmd 进程创建的 python 子进程
   大部分情况下这没问题，父进程会等待子进程结束再结束，当父进程被 kill 子进程也会被 kill

   但执行下述语句启动的 flask 进程无法被 kill 掉（具体原因不明）

   ```python
   p = subprocess.Popen('flask run -h 127.0.0.1 -p 8100', shell=True)
   import time
   time.sleep(5)
   p.kill()
   ```

### 失败原因

以下结论通过打开任务管理器监视不同代码运行时进程情况得出，可能理解有误

程序需要调用命令`python3 app.py -p 8080 -h 127.0.0.1`，创建一个 web 服务程序的子进程，并且可以在程序中将之关闭，尝试了很多办法都有问题

如下操作，windows 下在子进程中调用`os.system`时，共存在四个进程，父进程 A，父进程创建的子进程 B，进程 B 通过调用`os.system`创建的 cmd 进程 C，cmd 进程调用命令产生的子进程 D
父进程调用 kill，则进程 B 被 kill，进程 B 创建的子进程 C 也被销毁，然而 D 幸存

```python
class ProcessCocurrent(Process):
    # ...dosomething
    def run(self):
        os.system(self.cmd)
    # ...dosomething

```

如下操作，windows 下在子进程中调用`os.execvp`时子进程被覆盖，pid 改变，在父进程中无法 kill 子进程

```python
class ProcessCocurrent(Process):
    # ...dosomething
    def run(self):
        cmdli = self.cmd.split(" ")
        os.execvp(cmdli[0], cmdli)
    # ...dosomething

p = ProcessCocurrent()
p.start()
# 下述语句无法终止子进程
p.kill()
```

如下操作，直接调用`subprocess.Popen`,并设置`shell=True`，共存在三个进程，父进程 A，`subprocess.Popen(cmd, shell = True)`创建的 cmd 进程 B，cmd 进程创建的子进程 C。
`p.kill()`只能杀掉进程 B

```python
p = subprocess.Popen(cmd, shell = True)
p.kill()
```

### 正确做法

```python
p = subprocess.Popen(cmd)
# ...dosomething
p.kill()
```

linux 下还可以用

```python
pid = os.fork()
if pid == 0:
    os.execvp('python3','python3 app.py -p 8080 -h 127.0.0.1'.split(' '))
# ...dosomething
# 相当于kill -9 pid
os.kill(pid, 9)
os.waitpid(pid, 0)
```

### debug 模式

事实上，debug 模式就能监听文件变化并重启程序，由于没看文档绕了上述一个大圈子，可见看文档的重要性。

app.py 中开启 debug 模式

```python
# ...
if __name__ == "__main__":
    app.run(host, port, debug=True)
# or
app.debug = True
```

并且其实 flask 官方文档不建议用`python app.py`启动程序，而是建议通过`flask run -h host -p port`执行程序。

命令行开启 debug 模式

```bash
# windows
set FLASK_DEBUG=true&&flask run -h host -p port
# linux
export FLASK_DEBUG=true&&flask run -h host -p port
```

## 生成 tk 及转码

### express

```js
// server.js
const { create_tk, tu } = require("./create.js");
const utils = require("./util.js");
// 请求参数
const querys = {
  client: "webapp",
  sl: "zh-CN", //source language
  tl: "en", //target language
  hl: "zh-CN",
  dt: ["t", "at", "bd", "ex", "ld", "md", "qca", "rw", "rm", "sos", "ss"], // "t"],
  otf: "2",
  ssel: "3",
  tsel: "3",
  xid: "45662847,45683275",
  kc: "1",
};
const url = "https://translate.google.cn/translate_a/single?";
module.exports = function (app, multipartMiddleware) {
  app.all("*", function (req, res, next) {
    res.header("Access-Control-Allow-Origin", "*");
    res.header("Access-Control-Allow-Headers", "content-type");
    res.header("Access-Control-Allow-Methods", "PUT,POST,GET,DELETE,OPTIONS");
    res.header("Content-Type", "application/json;charset=utf-8");
    next();
  });
  app.get("/getstart", multipartMiddleware, (req, res) => {
    let { str = "" } = req.query;
    let query_str = "";
    const encodeResult = utils.checkEncode(str);
    if (encodeResult.indexOf("gbk") > -1) {
      str = utils.map2Unicode(utils.encodeISO(str)).join("");
    }
    // 可以是ascii编码的话，一定不存在中文
    if (encodeResult.indexOf("ascii") > -1) {
      querys.sl = "en";
      querys.tl = "zh-CN";
      // 不可以是ascii编码，直接视作中文
    } else {
      querys.tl = "en";
      querys.sl = "zh-CN";
    }
    querys.q = encodeURIComponent(str);
    // 组装get参数
    query_str = toqueryurl(querys) + tu(str);
    res.redirect(url + query_str);
  });
  app.get("/gettk", multipartMiddleware, (req, res) => {
    let { str = "" } = req.query;
    res.send(create_tk(str));
  });
};

// index.js，用以前写好的express模板
const express = require("express");
//返回一个函数
const multipart = require("connect-multiparty");
const applicationServer = require("./server.js");
function start(application) {
  //监听http请求
  var app = application || express();
  var bodyParser = require("body-parser");
  var multipartMiddleware = multipart(); //允许参数使用form-data格式,即contentType="multipart/form-data"
  var server;
  app.use(bodyParser.json({ limit: "1mb" })); //这里指定允许参数使用 json 格式,即contentType="application/json"
  app.use(
    bodyParser.urlencoded({
      extended: true,
    })
  ); //这里指定允许www-form-urlencoded格式,即contentType="application/x-www-form-urlencoded"

  applicationServer(app, multipartMiddleware);

  if (!application) {
    server = require("http").createServer(app);
    server.listen(7070);
    console.log("server start at loacalhost:7070");
  } else return app;
}
start();
```

然后 pm2 启动`pm2 start index.js -o log-file-path -e error-log-path -p pid-file-path -n app-name`
nginx 反向代理到`localhost:7070`

<details>
<summary>生成tk</summary>

```js
// 本来打算理解一下改成lua，觉得有点难，有闲情者可以试试
var lu = function (a, b, c) {
  for (var d = 0; c.test(b); ) {
    var e = c.lastIndex;
    e > d && a.push(b.substr(d, e - d));
    d = e;
  }
  b.length > d && a.push(b.substr(d));
};
(mu = / /g),
  (nu = /([?.,;:!][ ]+)|([\u3001\u3002\uff01\uff08\uff09\uff0c\uff0e\uff1a\uff1b\uff1f][ ]?)/g);
var ou = [0, 200],
  pu = {
    af: 1,
    ar: 1,
    bn: 1,
    bs: 1,
    ca: 1,
    cs: 1,
    cy: 1,
    da: 1,
    de: 1,
    el: 1,
    en: 1,
    eo: 1,
    es: 1,
    et: 1,
    fi: 1,
    fr: 1,
    gu: 1,
    hi: 1,
    hr: 1,
    hu: 1,
    hy: 1,
    id: 1,
    is: 1,
    it: 1,
    ja: 1,
    jw: 1,
    km: 1,
    kn: 1,
    ko: 1,
    la: 1,
    lv: 1,
    mk: 1,
    ml: 1,
    mr: 1,
    my: 1,
    ne: 1,
    nl: 1,
    no: 1,
    pl: 1,
    pt: 1,
    ro: 1,
    ru: 1,
    si: 1,
    sk: 1,
    sq: 1,
    sr: 1,
    su: 1,
    sv: 1,
    sw: 1,
    ta: 1,
    te: 1,
    th: 1,
    tl: 1,
    tr: 1,
    vi: 1,
    uk: 1,
    ur: 1,
    zh: 1,
    "zh-cn": 1,
    "zh-tw": 1,
  };
var qu = function (a) {
  return function () {
    return a;
  };
};
ru = function (a, b) {
  for (var c = 0; c < b.length - 2; c += 3) {
    var d = b.charAt(c + 2);
    d = "a" <= d ? d.charCodeAt(0) - 87 : Number(d);
    d = "+" == b.charAt(c + 1) ? a >>> d : a << d;
    a = "+" == b.charAt(c) ? (a + d) & 4294967295 : a ^ d;
  }
  return a;
};
su = null;
tu = function (a) {
  if (null !== su) var b = su;
  else {
    b = qu(String.fromCharCode(84));
    var c = qu(String.fromCharCode(75));
    b = [b(), b()];
    b[1] = c();
    b = (su = window[b.join(c())] || "") || "";
  }
  var d = qu(String.fromCharCode(116));
  c = qu(String.fromCharCode(107));
  d = [d(), d()];
  d[1] = c();
  c = "&" + d.join("") + "=";
  d = b.split(".");
  b = Number(d[0]) || 0;
  for (var e = [], f = 0, g = 0; g < a.length; g++) {
    var h = a.charCodeAt(g);
    128 > h
      ? (e[f++] = h)
      : (2048 > h
          ? (e[f++] = (h >> 6) | 192)
          : (55296 == (h & 64512) &&
            g + 1 < a.length &&
            56320 == (a.charCodeAt(g + 1) & 64512)
              ? ((h = 65536 + ((h & 1023) << 10) + (a.charCodeAt(++g) & 1023)),
                (e[f++] = (h >> 18) | 240),
                (e[f++] = ((h >> 12) & 63) | 128))
              : (e[f++] = (h >> 12) | 224),
            (e[f++] = ((h >> 6) & 63) | 128)),
        (e[f++] = (h & 63) | 128));
  }
  a = b;
  for (f = 0; f < e.length; f++) (a += e[f]), (a = ru(a, "+-a^+6"));
  a = ru(a, "+-3^+b+-f");
  a ^= Number(d[1]) || 0;
  0 > a && (a = (a & 2147483647) + 2147483648);
  a %= 1e6;
  return c + (a.toString() + "." + (a ^ b));
};
module.exports = { tu };
```

</details>

### 编码转换

可应用于此情况：逐字节读取 gbk 字符串，并转化为 unicode

编码转换一开始找了一堆转换函数，各种各样，我是实在不明白网上那些 utf82gbk,gbk2utf8 的代码到底是怎么回事，他们自己用过吗？
最后发现思路出了问题，unicode 和 gbk 字符集是人定义的，没考虑兼容,不可能存在函数经过一定处理转换两者，一开始搜的不应该是‘gbk 转 unicode’,而应该是 gbk unicode 映射表，一下就找到想要的答案。

[映射关系在此，cp 的意思是 code page](http://www.unicode.org/Public/MAPPINGS/VENDORS/MICSFT/WINDOWS/)
[这里是 code page 对应的编码格式](https://docs.microsoft.com/zh-cn/windows/win32/intl/code-page-identifiers)

gbk 编码范围是单字节 0x00~0x80,双字节高位 0x80~0xfe，低位不能是 0x7f（0x7f 为退格）

```js
+(function () {
  //引入映射关系
  const codemap = require("./codemap.js");
  //测试字符串
  const str = "asdas¼Ó+Ò»¶þ aÈýËÄÎÊÁùÆßasd\r\n";
  const utils = {
    // gbk单字节字符码点不大于0x80
    // 双字节字符首字节一定大于0x80
    // 匹配到一个字节大于0x80且（已知其前一个字节不大于0x80||已知其前一个字节属于一个双字节字符的第二个字节）
    // 那么这个字节一定是gbk双字节字符的首字节（编码格式限定在"iso-8859-1", "gbk", "unicode", "ascii"）
    // 逐字节读取转化为双字节gbk的二进制码
    encodeISO(str) {
      let arr = [];
      for (let i = 0; i < str.length; i++) {
        let num = str[i].charCodeAt(0);
        if (num > 0x80) {
          num <<= 8;
          i++;
          num += str[i].charCodeAt(0);
        }
        num = num.toString(16);
        arr.push(
          "0x" + (num.length > 2 ? num : num.padStart(2, "0")).toUpperCase()
        );
      }
      return arr;
    },
    // gbk映射到unicode
    map2Unicode(arr) {
      arr = arr.map((item) => String.fromCodePoint(parseInt(codemap[item])));
      return arr;
    },
    // 简单地区分几种编码
    checkEncode(arr) {
      let encoding = ["iso-8859-1", "gbk", "unicode", "ascii"];
      let flag = 1;
      //for of循环能正确识别大于等于二字节的unicode字符
      for (let chr of arr) {
        const code_point = chr.codePointAt(0);
        //unicode 存在>0xff
        if (code_point > 0xff) {
          return ["unicode"];
        }
        //gbk不存在低位等于0x7f的多字节字符
        else if (code_point == 0x7f && flag) {
          flag = false;
          if (encoding.indexOf("gbk") > -1)
            encoding.splice(encoding.indexOf("gbk"), 1);
        }
        //ascii 只有<0x80
        //unicode同样未定义0x81-0xff部分
        else if (code_point > 0x80) {
          flag = !flag;
          if (encoding.indexOf("ascii") > -1)
            encoding.splice(encoding.indexOf("ascii"), 1);
          if (encoding.indexOf("unicode") > -1)
            encoding.splice(encoding.indexOf("unicode"), 1);
        }
        //iso-8859-1 单字节字符 0x00-0xff 既有<0x40 也有>0x80
        //gbk 存在>0x80 至少>=0x40
        else if (code_point < 0x40) {
          if (encoding.indexOf("gbk") > -1)
            encoding.splice(encoding.indexOf("gbk"), 1);
        } else {
          flag = false;
        }
      }
      return encoding;
    },
  };
  // console.log(module)
  // console.log(process.argv)
  if (process.argv.indexOf(module.filename) > -1) {
    let result1 = utils.encodeISO(str);
    let result2 = utils.map2Unicode(result1);
    console.log(result1);
    console.log(result2);
  } else {
    module.exports = utils;
  }
})();
```

## rainmeter 皮肤

```ini
[Rainmeter]
Update=100
AccurateText=1

[Metadata]
Name=
Author=
Information=
Version=
License=Creative Commons Attribution - Non - Commercial - Share Alike 3.0

[Variables]
UserInput="请输入"
BackgroudColor=0,0,0,200
FontColor=255,255,255
# 记录点，用于控制计时器
BreakPoint=0
TimerRange=5

[MStyle]
X = 0
Y = 30r
SolidColor=#BackgroudColor#
FontColor=#FontColor#
Padding=10,10,10,10

[InputStyle]
X=0
Y=3
W=210
H=15
Padding=5,5,5,5
SolidColor=#BackgroudColor#
FontColor=#FontColor#

# 倒计时器TimerMeasure
# 逻辑如下
# 时钟Clock循环从0到TimerRange变化
# 每次启动TimerMeasure计时器，用BreakPoint记录启动的时刻
# Clock再次等于BreakPoint时计时倒计时停止
# 根据Clock的值和BreakPoint的值计算计时器经过的时间
# 例如BreakPoint为3，Clock为5由于Clock递增可知过去了Clock-BreakPoint=2秒，剩余TimeRange+BreakPoint-Clock秒
# 若BreakPoing为3,Clock为1可知Clock循环了一次回到了1
# 此时经过的时间可用TimerRange-BreakPoint+Clock得出，剩余BreakPoint-Clock秒
[Clock]
Measure=Calc
Formula=(Clock+1)%(#TimerRange#+1)
DynamicVariables=1
UpdateDivider=10

[TimerMeasure]
Measure=Calc
Formula=#BreakPoint#>Clock?#BreakPoint#-Clock-1:#TimerRange#+#BreakPoint#-Clock
DynamicVariables=1
UpdateDivider=10
IfEqualValue=0
IfEqualAction=[!PauseMeasure "TimerMeasure"][!HideMeter "MyMeter2"]
Paused=1

# 另一个倒计时器，不采用此方法的原因为，无法重置时间，即每次必须倒数完设定的秒数才能重新开始倒数
# 上一个倒计时器可以通过重设breakPoint重置
; [TimerMeasureT]
; Measure=Calc
; Formula=TimerMeasureT>0?TimerMeasureT-1:#TimerRange#
; IfEqualValue=0
; UpdateDivider=10
; DynamicVariables=1
; IfEqualAction=[!HideMeter "MyMeter2"][!PauseMeasure "TimerMeasureT"]
; Paused=1

[JsonMeasure]
Measure=Script
ScriptFile=script.lua
DefaultValue="N/A"
UpdateDivider=-1
OnChangeAction=[!ShowMeter "MyMeter2"][!UpdateMeter "MyMeter2"]


[WebMeasure]
Measure=Plugin
Plugin=WebParser
URL=http://localhost:7070/getstart?str=[#UserInput]
RegExp=(?siU)^(.*)$
StringIndex=1
ClipString=1;自动换行
Substitute="":"N/A"
UserAgent=Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:73.0) Gecko/20100101 Firefox/73.0
FinishAction=[!UpdateMeasure "JsonMeasure"][!PauseMeasure "WebMeasure"]
UpdateDivider=-1
; 动态加载变量
DynamicVariables=1
Paused=1


[MeasureInput]
Measure=Plugin
Plugin=InputText.dll
X=5
Y=8
W=160
H=15
Padding=5,5,5,5
SolidColor=#BackgroudColor#
FontColor=#FontColor#
FontFace=浪漫雅圆
FontSize=8
FocusDismiss=1
Command1=[!SetVariable UserInput "$UserInput$"][!ShowMeter "DefaultText"][!UpdateMeter "DefaultText"][!UnpauseMeasure "WebMeasure"][!CommandMeasure WebMeasure Update][!UpdateMeasure "WebMeasure"]
DefaultValue=#UserInput#
OnDismissAction=[!ShowMeter "DefaultText"][!UpdateMeter "DefaultText"]
Container=DefaultText
UpdateDivider=-1


[ImageMeter]
Meter=IMAGE
ImageName=1.png
Container=ContainerMeter
UpdateDivider=-1

[DefaultText]
Meter=STRING
MeterStyle=InputStyle
DynamicVariables=1
Text=#UserInput#
LeftMouseUpAction=[!HideMeter "DefaultText"][!RainmeterPluginBang "MeasureInput ExecuteBatch 1"]
Container=ContainerMeter
UpdateDivider=-1

[MyMeter2]
Meter=String
MeasureName=JsonMeasure
MeterStyle=MStyle
Container=ContainerMeter
UpdateDivider=-1
Hidden=1
; OnUpdateAction=[!UnpauseMeasure "TimerMeasureT"][!SetVariable TimerRange 4]
OnUpdateAction=[!SetVariable BreakPoint [Clock]][!UnpauseMeasure "TimerMeasure"][!SetVariable TimerRange 4]

[ContainerMeter]
Meter=Shape
Shape=Rectangle 0,0,500,1000,8 | Fill Color 255,47,47,255 | StrokeWidth 0
UpdateDivider=-1

```

lua 脚本

```lua
function Initialize()
    json = dofile(SKIN:MakePathAbsolute('json.lua'))
    encoding = dofile(SKIN:MakePathAbsolute('encoding.lua'))
end

function concat_means(means)
    local result = ""
    for key, mean in ipairs(means) do
        result = result .. mean[1] .. ": "
        for key2, mean_content in ipairs(mean[2]) do
            result = result .. mean_content .. ", "
        end
        result = result .. "\n"
    end
    return result
end

function Update()
    local jsonStr = SKIN:GetMeasure('WebMeasure'):GetStringValue()
    local user_input = SKIN:GetVariable("UserInput", "N/A")
    if user_input == "N/A" or user_input == "请输入" then
        return user_input
    end
    if not (jsonStr == "N/A") then
        local json_obj = json.decode(jsonStr)
        local translate_group = json_obj[2]
        print(json_obj[2], json_obj[3])
        local result = ""
        local translate = "翻译: " .. json_obj[1][1][1] .. "\n"
        -- 翻译
        local source_text = "原文: " .. json_obj[1][1][2] .. "\n"
        -- 原文
        local pinyin = "拼音: " .. json_obj[1][2][4] .. "\n"
        -- 拼音
        result = result .. "\n"
        if not (translate_group == nil) then
            for key, group in ipairs(translate_group) do
                local group_name = group[1]
                local group_member = group[2]
                local group_member_means = group[3]
                result = result .. group_name
                result = result .. "\n"
                result = result .. concat_means(group_member_means)
                result = result .. "\n"
            end
        end
        result = source_text .. pinyin .. translate .. result
        return result
    else
        return jsonStr
    end
end
```
