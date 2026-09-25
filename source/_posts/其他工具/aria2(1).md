---
title: 利用aria2下载文件
tags: 
- aria2
- bash
date: 2018/04/12
---
# 前言
在长时间使用迅雷下载后，产生替换下载工具的想法，然而不得不承认，迅雷是具有其一定的优势的，要找到替代产品可不容易，这次要提到的aria2是本人在网上找到的一个不错的替代品之一。此文主要讲述Windows上的使用方法。<!--more-->

# 内容

## aria2优势所在

1.  轻便
    作为一款命令行下载程序，aria2的大小总共只有不到5兆，相较于迅雷。。。,完全无法比较

2.  功能齐全
    aria2比较类似于linux上的wget命令,支持ftp下载，http下载,种子，磁链等，除了专门的迅雷下载链接，其余迅雷能下的他都能下

3.  可能性
    类似于许多优秀的工具，aria2是可扩展的，可以配合插件，或者配合一些库使用，也有不少aria2的受益用户主动为其编写插件，因而其功能或表现可以得到极大的扩展，这便是开源的优势所在

## 安装

可以从github上下载，[下载链接在此](https://github.com/aria2/aria2/releases/tag/release-1.33.1)。

下载下来的文件是个压缩包，解压后无需安装即可使用。

下载的压缩包内容如下

![图片无法显示](../image/default.png)

双击aria2c.exe可以看到一闪而过的dos窗口，有过编程经验的人的人都能明白这是dos界面的程序，一闪而过是因为输出完后程序运行结束，dos窗口被关闭所致。因而我们需要在命令行内执行该可执行程序。
打开命令行，我们要用到第二个常识，一般的命令行程序运行时加上-h或-help或--help便是输出帮助信息，我们只需逐个尝试，很幸运aria2c的帮助是-h,我们能看见aria2c拥有许多的参数，aria2c将其参数分类为几种标签，有#base(-h默认)，#help(-h#help之后类推),#advanced,#rpc,#http,#all等等.我们只要知道如何输出帮助，之后参数有什么作用可以慢慢看。

由于我并不是很熟悉windows命令行操作，我将所有参数的解释复制在了文本中方便查找。
由于不是所有看这篇文章的都知道，提一下，可以在浏览器，word,文本编辑器等许多浏览文档的地方用ctrl+F查找文本

为了方便使用我们将aria2c.exe所在路径添加到环境变量path中,例如我的aria2c.exe路径是D:\aria2\aria2c.exe,那么，我将D:\aria2加入path中。（如何添加环境变量请自行百度）
可以再加一个ARIA2_HOME环境变量（变量名其实无所谓），值设为D:\aria2(视情况而定)，之后提供的现成的配置方案中会用到此参数。


## 开始使用

通过帮助信息我们可以看到一般的下载只需要如下格式便可
```bash
aria2c [option] url
aria2c -t (path of torrent file)
```
但是通过这样的方式并不方便，我们每次下载都需要许多参数，来控制存放路径，并发下载数，最大下载速度等等，重复操作过多
因而我们需要采用配置文件的方法，配置文件的编写，需要查看aria2c的帮助信息，学习各参数的作用，对于拥有巨量参数的aria2c来说，这并不是一件容易的事，幸而网上有已经配置好的配置文件可以直接拿来用。
自行百度***aria2c配置***，查找现成的配置文件,
配置文件通过
```bash
--config-path=path
```
参数载入，具体如何配置，我下文会给出示例文件的链接

## rpc下载

百度了一下，rpc是所谓的远程方法调用，aria2c开启rpc功能后（开启方法:命令行后加参数--rpc-enable=true或配置文件中rpc-enable=true)，不会自动关闭，会一直监听指定端口，供用户调用下载相关的函数。下载相关数据信息通过json传输。网上有配合aria2c rpc服务的aria2的webui.webui我在下文提供的文件中会给出
rpc相关的参数可通过
```bash
aria2c -h#rpc
```
来查看

我便是采用了rpc+webUi的方法来下载文件。

我的文件结构如图

![图片无法显示](../image/file_struct.png)

另在windows中,启用rpc服务时，参数-D，或者配置文件中的daemon=true并不能使程序在后台运行，本人为了达成后台运行搜索了许多办法，start /b指令，windows定时任务，mfc程序后台调用系统命令且不显示窗口等等都不能达成目的，最后在别人的博客上看到了vbs脚本启动的方法才行。启动代码如下。
```bash
CreateObject("WScript.Shell").Run "aria2c.exe --conf-path=%ARIA2_HOME%/conf/config.conf --dir=%ARIA2_HOME%/downloads --log=%ARIA2_HOME%/log/aria2.log --input-file=%ARIA2_HOME%/session/aria.session --save-session=%ARIA2_HOME%/session/aria.session",0
//此处用到了之前设置的ARIA2_HOME环境变量，代码根据别人文件改动而来，必须先设置好环境变量
```
将蕴含以上代码的文本文件命名为aria2c_run.vbs,并与aria2c.exe放于同一目录下，由于之前设置了环境变量，现在我们可以在命令行中输入aria2c_run来运行aria2c服务了，或者你可以把aria2c_run.vbs文件置于桌面，点击即可启动，之后可以通过webui来确定是否启动成功。

## 添加trackers

aria2c不添加trackers前，对bt下载是比较慢的。
tracker我并不理解是什么，似乎是某种p2p链接中继或者高一级的服务节点之类的吧
添加方法
```bash
//配置文件
bt-tracker=url1,url2,url3......
```
[此处有其他人提供的trackers链接](https://github.com/ngosang/trackerslist)（github上的）

<!-- # 附
本人的配置文件+主程序等，解压后，按上文配置好环境变量即可使用，配置文件改自网上现成的文件，其中还有help.txt(所有参数的英文解释),webui.html(重定向至某一webui界面)：
度盘：[https://pan.baidu.com/s/1wo5ndPkT9VFYd4ZT-tXWsQ](https://pan.baidu.com/s/1wo5ndPkT9VFYd4ZT-tXWsQ) 密码：aew0 -->

