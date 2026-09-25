---
title: 当前目录下打开管理员命令行
tags:
  - 命令行
  - windows
data: 2020/04/13
---

# 前言

win10 当前目录下打开管理员命令行比较麻烦。一种是`win+R`然后输入`cmd`接着按住`ctrl+shift`回车，或直接找到`cmd.exe`右键以管理员身份运行，再在弹出的窗口里`cd /d path`到当前路径

`win+x`现在只能打开`powershell`，虽然很多说`powershell`比`cmd`好用的，但还是更习惯`cmd.exe`。

打开`cmd`后如果需要转到一个比较长的路径的话，不管是复制粘贴还是**手输+`Tab键`**都挺麻烦的，因此想要一个更简单的最好鼠标点击两下就完成的操作。

<!-- more -->
<!--
details>
<summary>powershell的使用不习惯原因</summary>

我第一次用的时侯，没好好看文档，碰到不少不习惯的地方，也成为我不喜欢powershell的原因

第一次使用时，用`cd /d`命令，不能用，就想看帮助，`cd help`，`cd /?`，`cd --help`，`cd -h`，都不能用，输入`help`给出`Get-Help`的帮助，以为出错了，然后我试了一下`help cd`，显示`Set-Location`的帮助，才知道`cd`，`help`等命令应该是为了习惯了用命令行的人转用`powershell`给`Set-Location`，`Get-Help`命令起的别名，问题是这些命令的使用方法完全不同，这种别名只会造成迷惑吧？

命令行下输入`help`命令显示全部可用命令简单描述，powershell上只能看到`get-help`命令的用法，还得对每个命令`help <cmdlet>`才能看到每个命令有什么用。

使用时又发现有些命令和命令行相同，输`help attrib`的时候给出现不相关的命令列表，又用`attrib /?`发现能输出，又尝试了很多命令，最后试了一下通配符`help *`列出了全部命令列表，但没有简单描述各命令的作用。

另外powershell中命令写起来比较复杂，例如原来的`dir /s /b /aohs`就得写成`dir -depth num -name -attributes system,hidden,offline,reparsepoint`

还有powershell那个红色的报错配合背景色看着别扭。

我觉得如果简单地写一些处理命令，`powershell`并不合适，命令长且复杂，一些被视为优点的新特性也用不到。`powershell`应该是适合那些更专业的人，需要做许多比较复杂的操作的。
</details
-->

# 方案

我之前一直用的是如下方案：

下面脚本复制自`stack overflow`，做了些改动,命名为 sudo.bat 放在 system32 文件夹下,
并且向注册表中添加`open admin here`右键菜单项，command 项值为`sudo.bat "%V"`，脚本的逻辑是

1. 执行一条需要管理员权限的命令，通过 errorlevel 来判断执行是否成功如果成功直接执行 gotAdmin
2. 如果失败，执行 UACPromt 生成临时的 vbs 脚本文件，在脚本中通过 UAC 请求管理员（会有弹窗问向用户询问授权，都见过的那个）权限后再执行一次 sudo.bat，此时 sudo.bat 会执行 gotAdmin 部分的代码
3. gotAdmin 的代码 cd 到参数指向的路径

```shell
:: BatchGotAdmin
:-------------------------------------
@echo off
REM --> Check for permissions
REM -->nul 2>&1 "%SYSTEMROOT%\system32\icacls.exe" "%SYSTEMROOT%\system32\config\system"
if "%1" == "1" (
    goto gotAdmin
)
# 此命令需要管理员权限
netstat -naob>>"%temp%\getadmin.txt"

REM --> If error flag set, we do not have admin.
if '%errorlevel%' NEQ '0' (
    echo Requesting administrative privileges...
    goto UACPrompt
) else ( goto gotAdmin )

:UACPrompt
    echo Set UAC = CreateObject^("Shell.Application"^) > "%temp%\getadmin.vbs"
    echo args = "" >> "%temp%\getadmin.vbs"
    echo For Each strArg in WScript.Arguments >> "%temp%\getadmin.vbs"
    echo args = args ^& strArg ^& " "  >> "%temp%\getadmin.vbs"
    echo Next >> "%temp%\getadmin.vbs"
    echo UAC.ShellExecute "%~f0", args, "", "runas", 1 >> "%temp%\getadmin.vbs"
    REM type %temp%\getadmin.vbs
    REM 命令行执行
    "%temp%\getadmin.vbs" 1 "%cd%" %*
    echo %*
    exit /B

:gotAdmin
    if exist "%temp%\getadmin.vbs" ( del "%temp%\getadmin.vbs" )
    echo "%%1: %3,%~dp3" "%%2: %2" "%%~f1: %~f1"
    if "%3"=="" ( CD /D "%~f2" ) else ( CD /D "%~df3" )
    cmd /k
:--------------------------------------
```

## 问题

然而脚本有一个缺点：用 setx 命令把带空格路径添加到系统变量时，通过 sudo.bat 在当前路径打开管理员命令行发现带空格的路径会被识别为两个参数

<details>
<summary>失败尝试</summary>

一开始不知道哪出了问题，就想寻找替代方案打开管理员命令行，runas 因此登场（其实 runas 之前试过每次都以失败告终此已经是第 N 次尝试）。

## runas 尝试

![图片无法显示](../image/runas/找不到路径.png)

runas 的使用格式`runas /user:domain\username "要执行的命令"`，其他的可在命令行下`runas /?`查看。
可以不输入 domain，不输入时为当前主机
网上都说`runas /user:admin "你的命令"`或者`runas /user:administrator "你的命令"`就行。
然而至少 win10 下并不是这样：
执行`runas /user:我的当前用户名（有管理员身份） "cmd /k"`输入密码打开新的 cmd 的窗口发现并没什么用，得到的窗口还是普通用户权限，试了另一个由管理员账号的用户名，发现还是普通用户权限。

![图片无法显示](../image/runas/无效.png)

按网上教的直接执行`runas /user:administrator "cmd /k"`，也要求输入 administrator 用户的密码，但是我根本没有设置这个用户的密码。我电脑唯一设置的密码只有我的当前用户的密码，输入这个密码提示用户名和密码错误，不输入直接回车提示 runas 不能不输入密码执行。。。

![图片无法显示](../image/runas/密码不正确.png)

另外 runsa 不管输入什么用户名都会提示输入密码，而不会检查用户是否存在，也就是说 administrator 用户不一定存在，需要查看我的电脑上到底有哪些用户，哪些用户是管理员，于是打开了控制面板，系统用户、管理其他用户，发现只有一个当前用户，但是我明明记得之前还有一个用户名不同的本地用户，一个是用邮箱注册的 microsoft 官网用户，都可以用 runas 登录，和当前账户的性质肯定不同，难道又是什么别名。

**能看到的用户(另一个是我后来加的)**

![图片无法显示](../image/runas/系统用户.png)

我还发现控制面板改不了用户密码了，只能到 win10 的设置里改当前用户密码，改不了其他用户的密码。

![图片无法显示](../image/runas/当前用户.png)

另外计算机管理里面没有了用户和组管理，根据百度用 mmc 控制台打开也没有用，直接提示不能用。
根本看不到具体有哪些用户了，真的是越更新越难用。

![图片无法显示](../image/runas/mmc.png)

然后再用`net user`命令查看用户，发现能看到本地，但看不到不是本地的当前用户，而且还多了一个奇怪的名字是我邮箱开头五个字母的奇怪用户，我不记得我创建过，以及一个默认的 guest 用户。越看越糊涂。这两个查看到的用户都明显不全。

![图片无法显示](../image/runas/netuser.png)v

### runas 结论

最后我通过**修改文件属性-->安全-->高级-->添加**里的**搜索用户**功能终于搜到了比较全面的用户列表。

![图片无法显示](../image/runas/用户列表.png)

确实存在 administrator 这个用户，但是不知道这个用户的密码，于是又百度，得到的解答是**必须得为执行`runas`命令的用户设置一个密码**，但是像上文说的，win10 上我无法更改其他用户的密码，甚至无法在设置里看到这个 administrator 用户。。。并且以 administartor 身份执行 runas 命令打开的命令行窗口不一定就有管理员权限，毕竟我的当前用户也有管理员身份，因此最终得到结论 win10 上 runas 以管理员身份运行命令不可行（至少我目前这么低的水平做不到），顶多是以其他用户身份普通用户权限执行命令执行。

</details>

## 解决问题

最终还是花了不少时间改了 sudo.bat

### 问题解析

1. 命令设置为`sudo.bat 当前路径`时，如果当前路径是带空格的路径，会被当成两个参数，无法正确处理。
2. 如果命令为`sudo.bat "当前路径"`参数的引号不会被去除，带引号的参数传到`vbs`文件里引号会被去除，因此`bat`文件中参数是路径时最好用`~`去除引号（如`%~1`），否则容易引起混乱，这个问题导致我自己写`test.bat`以及`test.vbs`调试时浪费了许多时间。
3. `vbs`文件中用`ShellExecute`执行`bat`文件时实际上执行的是`cmd /c 文件.bat 参数`，因此即使`bat`文件执行出错会立即退出窗口，不方便测试查看错误。

### 最终结果

```shell
:: BatchGotAdmin
:-------------------------------------
@echo off
@REM Check for permissions
@REM nul 2>&1 "%SYSTEMROOT%\system32\icacls.exe" "%SYSTEMROOT%\system32\config\system"
@REM If error flag set, we do not have admin.
@REM netstat -naob需要管理员权限

@REM 启用变量延迟
setlocal enabledelayedexpansion
call:increase %~2

@REM >nul似乎是用于重定向回显信息及错误提示到空设备（即屏蔽不显示），nul代表空设备
netstat -naob>nul
if '%errorlevel%' NEQ '0' (
    @REM 由于不太清楚errorlevel机制，限制尝试次数，避免errorlevel值出现意外情况
    if !trytime! GTR 3 (
        echo get privileges failed
        pause
        exit /B
    ) else (
        echo Requesting administrative privileges...
        goto UACPrompt
    )
) else (
    echo got administrative privileges
    goto gotAdmin
)

:UACPrompt
    echo Set UAC = CreateObject^("Shell.Application"^) > "%temp%\getadmin.vbs"
    echo args = "/k %~f0 " >> "%temp%\getadmin.vbs"
    echo For Each strArg in WScript.Arguments >> "%temp%\getadmin.vbs"
    @REM ^为bat文件转义符，""""在vbs中代表一个双引号字符
    @REM 字符串拼装
    echo args = args ^& """" ^& strArg ^& """" ^& " "  >> "%temp%\getadmin.vbs"
    echo Next >> "%temp%\getadmin.vbs"
    @REM echo msgbox(args) >> "%temp%\getadmin.vbs"
    @REM ShellExecute第一个参数不是.exe可执行文件时，实际上会执行cmd /c sudo.bat ...
    @REM 为了避免这一情况选择cmd.exe作为可执行文件
    echo UAC.ShellExecute "cmd", args, "", "runas", 1 >> "%temp%\getadmin.vbs"
    %temp%\getadmin.vbs "%cd%" !trytime!
    @REM exit退出，/B代表退出脚本而不是退出cmd
    exit /B

:gotAdmin
    if exist "%temp%\getadmin.vbs" ( del "%temp%\getadmin.vbs" )
    @REM echo "%%~f1: %~f1"
    title %~f1
    cmd /k CD /D "%~f1"

@REM trytime值加1
:increase
    set trytime=%~1
    @REM 单引号避免空值引起错误
    if '!trytime!' == '' (
        set trytime=0
    )
    set /a trytime=!trytime!+1
    exit /B
:--------------------------------------
```

**注意**

1. 使用路径作为参数时最好用`~`去除双引号，否则可能出问题，如`%~f1`。
2. 不管是否有双引号，`cmd /c`或`cmd /k`无法正确执行路径中带空格的文件
