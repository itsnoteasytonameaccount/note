---
title: windows上安装Emscripten
date: 2020/12/15 20:56
---

# 前言

因为想试试 webassembly ，需要安装 Emscripten，但是安装出了蛮多问题的

## 步骤

先用 git 下载 EmscriptSDK，这步比较简单

```bash
git clone https://github.com/emscripten-core/emsdk.git
```

然后通过 emsdk 安装 Emscripten

```bash
emsdk install latest
```

## 问题

1. node一直下载失败导致安装步骤退出

这个问题令我很疑惑，我在同样的地址下载过 node 的压缩包，为什么在命令行就下载不下来呢。
并且这个暂且不提，我本地明明有三个版本的 node，用 nvm-windows 管理，其他的软件都能检测到，偏偏emsdk这玩意检测不到，还要给我下，令有些喜欢钻牛角尖我非常不爽。

**解决方案**: 本地没有 Node 环境可以用本文[参考](#参考)中引用的链接中的方法，有 node 环境可以改代码。

只能改 emsdk.py 代码来跳过：

```python
# 1871行左右，跳过本地已有的环境
  def install_tool(self):
    # Avoid doing a redundant reinstall of the tool, if it has already been installed.
    # However all tools that are sourced directly from git branches do need to be
    # installed every time when requested, since the install step is then used to git
    # pull the tool to a newer version.

    # 判断是否安装安装了的跳过
    print("name is: "+self.name)

    def is_installed(name):
	#   不想安装列表
      installed_list = ["node","python","java"]
      for installed in installed_list:
        if installed in name:
          return True
      return self.is_installed()

    if is_installed(self.name) and not hasattr(self, 'git_branch'):
      print("Skipped installing " + self.name + ", already installed.")
      return True

# 1243行左右，指定node地址
def emscripten_npm_install(tool, directory):
  # node_tool = find_latest_installed_tool('node')
  # if not node_tool:
  #   print('Failed to run "npm ci" in installed Emscripten root directory ' + tool.installation_path() + '! Please install node.js first!')
  #   return False
  print("directory is:"+directory)
  # node_path = os.path.join(node_tool.installation_path(), 'bin')
  node_path = "D:\\environment\\node"
  npm = os.path.join(node_path, 'npm' + ('.cmd' if WINDOWS else ''))
  env = os.environ.copy()
  env["PATH"] = node_path + os.pathsep + env["PATH"]
```

2. python版本问题

安装过程中又提示一个.py文件运行出错，浪费了一些时间，想起本地安了两个python，应该是用了python2的关系。
就想着给python也有像node的nvm那样管理和切换版本的就好了，因此创建了如下文件

__pyvm.bat__ 
用来切换版本的命令
python都放在D:\environment

```bash

@echo off

set environment_path=D:\environment
if '%1' == 'list' (
    dir /b %environment_path%|findstr /i python
) else ( 
    if exist %environment_path%\%1. (
        switch_python_version.vbs %environment_path%\%1
    ) else (
        echo "command unrecognized!"
    )
)
```

__switch_python_version.vbs__

setx指令需要管理员权限所以用vbs的UAC对象申请

```vbs
args = "/c setx /m PYTHON_HOME "
Set UAC = CreateObject("Shell.Application")

For Each strArg in WScript.Arguments
    args = args & """" & strArg & """" & " "
Next

msgBox(args)

UAC.ShellExecute "cmd", args,"" , "runas", 1
```

之后在环境变量中添加`%python_home%\scripts`和`%python_home%`，就可以自由通过`pyvm`切换版本了。

## 成功

安装成功后就输出如下

```bash
Installing SDK 'sdk-releases-upstream-37fc7647c754ac9a28ad588c143b82286de0ef71-64bit'..
name is: node-12.18.1-64bit
Skipped installing node-12.18.1-64bit, already installed.
name is: python-3.7.4-pywin32-64bit
Skipped installing python-3.7.4-pywin32-64bit, already installed.
name is: java-8.152-64bit
Skipped installing java-8.152-64bit, already installed.
name is: releases-upstream-37fc7647c754ac9a28ad588c143b82286de0ef71-64bit
Skipped installing releases-upstream-37fc7647c754ac9a28ad588c143b82286de0ef71-64bit, already installed.
directory is:D:/projects/webnext/emsdk\upstream\emscripten
Running post-install step: npm ci ...
Running post-install step: npm install google-closure-compiler-windows
Done running: npm ci
Done installing SDK 'sdk-releases-upstream-37fc7647c754ac9a28ad588c143b82286de0ef71-64bit'.
```

接着
```bash
emsdk activate latest
# 可附加--permanent永久修改
```
初始化环境


因为跳过了一些安装所以有如下提示，关系不大

```bash
Warning: The SDK/tool 'node-12.18.1-64bit' cannot be activated since it is not installed! Skipping this tool...
Warning: The SDK/tool 'python-3.7.4-pywin32-64bit' cannot be activated since it is not installed! Skipping this tool...
Warning: The SDK/tool 'java-8.152-64bit' cannot be activated since it is not installed! Skipping this tool...
```

最后测试一下成功没
```bash
> emcc -v
# 输出
emcc (Emscripten gcc/clang-like replacement + linker emulating GNU ld) 2.0.10
clang version 12.0.0 (Cswircachegitchromium.googlesource.com-external-github.com-llvm-llvm--project 445289aa63e1b82b9eea6497fb2d0443813a9d4e)
Target: x86_64-pc-windows-msvc
Thread model: posix
InstalledDir: D:/projects/webnext/emsdk/upstream/bin
shared:INFO: (Emscripten: Running sanity checks)
```

# 参考

1. [《windows下和linux 安装emscripten》](https://www.it610.com/article/1281884758007103488.htm)
2. [webassembly开发者引导](https://www.wasm.com.cn/getting-started/developers-guide/)