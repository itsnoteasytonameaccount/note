---
title: 星际争霸2api库编译记录
tags:
- c++
date: 2021/08/18
---

据了解，rts游戏以前常被用来进行深度学习方面的研究。

出于个人兴趣，本人进行了一些简单尝试，找了星际争霸的api，编写程序调用api自动对战。本文记录经验。

<!-- more -->

# 不同的库

官方github上列出了不同语言环境下的各类工具，初看时有点眼花缭乱
因此先简单介绍一下各种工具

## pysc2

这个是社区作者提供的python版本api调用库，为你编写自己的机器人提供python环境，文档不齐全，https://github.com/deepmind/pysc2，网上相关介绍也不多，这是我唯一找到的还算可以的[介绍博客](https://www.jianshu.com/p/9f7c3cd83bf0)

## s2client-api

[s2client-api](https://github.com/Blizzard/s2client-api)
这个是官方提供的c语言api调用库，文档是代码注释自动生成的，烂的一比，代码里还有bug一直没改

## s2client-proto

[s2client-proto](https://github.com/Blizzard/s2client-proto)
这里有接口相关的定义，以及资源汇总，不愿用官方提供的api调用库的，直接自己写调api的方法也可以，好像要用websocket调

# 准备环境

本文选用的是s2client-api

本人是windows系统，因此下文的都是windows下的操作，其他环境参考官方文档

官网已有介绍vs studio17下的配置方式，但出于某些原因实在不愿意用vs，因此自行使用命令行编译。

## 需要已安装

cmake，mingw64，星际争霸游戏本体

mingw64和cmake的安装只要到相应的网址下安装包（由于是很久之前装的，此处就不提供链接了）。

# 编译s2client-api库

1. 用`git clone`命令下载好s2client-api，clone时必须加`--recursive`参数将相关依赖都下下来

2. 进到s2client-api目录，创建build文件夹

3. 在build文件夹打开cmake-gui，点击cmake-gui左下角configure生成配置选项后，根据需要勾选（一般不用修改）,之后点击generate生成makefile文件

![image](../image/sc2api/cmake-gui.png)

4. 最后在build文件夹下执行mingw32-make命令。

> 第四步可能会提示缺少一个EGL.h头文件，说实话我不知道这是什么，应该是图形相关头文件。我的解决方案是用everything搜索本地，正好本地的emscripten编译器目录下有这个头文件，本地没有的可能需要百度百度。

> 找到egl.h头文件后需要将这个头文件路径加入到`C_INCLUDE_PATH`环境变量中

windows下命令

```bash
set C_INCLUDE_PATH=xxx
```

5. 然后make，就可以在`build/bin`文件夹下找到对应的静态库了，如下

![图片无法显示](../image/sc2api/编译结果.png)

6. 编写程序时链接库文件，并引入相关头文件即可（下文有简单示例）

## 枚举数据设置问题

实际上不做任何修改的话，依赖上文操作编译出来的库编译出来的程序是会出问题的，具体原因是游戏程序的接口返回的数据标志，和代码中预期的数据标志不一致，导致程序和游戏进程间停止交互，需要修改源码

具体问题见如下链接
https://github.com/Blizzard/s2client-api/pull/306
https://github.com/Blizzard/s2client-api/issues/310
https://github.com/Blizzard/s2client-api/issues/315

需要进行的修改见如下链接

https://github.com/Blizzard/s2client-api/pull/306/commits/7e2cb8f2b2a49f188380f051cf5fcd72ca94540d
https://github.com/peter-cudmore/s2client-api/commit/0a79825bd1c6318debb56082b9142f4940b7d73c

# 编写自己的程序并编译

编写自己的程序请参看官方文档，

示例代码

```c++
#include <sc2api/sc2_api.h>
#include <iostream>
#include "sc2utils/sc2_manage_process.h"

using namespace sc2;

class Bot : public Agent
{
private:
    /* data */
public:
    virtual void OnGameStart() final
    {
        std::cout << "hello world" << std::endl;
    };
    virtual void OnStep() final
    {
    };
    virtual void OnUnitIdle(const Unit *myunit)
    {
        std::cout << "idle" << std::endl;
    };
    virtual void OnUnitCreated(const Unit *myunit)
    {
        std::cout << "created" << std::endl;
    };
};

int main(int argc, char *argv[])
{
    Coordinator coordinator;
    Bot bot;
    Bot bot2;
    // 必须，否则无法检测到游戏本体的路径
    if (!coordinator.LoadSettings(argc, argv))
    {
        std::cout << "load failed" << std::endl;
        return 1;
    }
    coordinator.SetRealtime(true);
    coordinator.SetParticipants({CreateParticipant(Race::Terran, &bot),
                                 CreateParticipant(Race::Terran, &bot2)});

    coordinator.LaunchStarcraft();
    coordinator.StartGame(sc2::kMapBelShirVestigeLE);

    while (coordinator.Update())
    {
    }
    return 0;
}

```

编译时需要引入的库如下（下面是vs code的task配置文件，应该不难看懂，其中`${}`包括的内容是vscode内置变量，用于读取源文件名或环境变量）

示例配置

```json
        {
            "label": "build folder",
            "group": "build",
            "type": "shell",
            "command": "g++",
            "args": [
                "${relativeFileDirname}/*${fileExtname}",
                "-static",
                "-L",
                "%LD_LIBRARY_PATH%",
                "-lsc2api",
                "-lsc2lib",
                "-lsc2utils",
                "-lsc2protocol",
                "-lprotobuf",
                "-lcivetweb",
                // 额外引入的库，civetweb依赖这个
                "-lws2_32",
                "-o",
                "${relativeFileDirname}.exe"
            ],
            "options": {
                "env": {
                    "CPLUS_INCLUDE_PATH": "${env:ENV_PATH}/s2client-api/include;${env:ENV_PATH}/s2client-api/build/generated;${env:ENV_PATH}/s2client-api/contrib/protobuf/src",
                    "LD_LIBRARY_PATH": "${env:ENV_PATH}/s2client-api/build/bin"
                }
            },
            "problemMatcher": "$gcc"
        },
```

# commandCenter

[CommandCenter](https://github.com/davechurchill/CommandCenter)
这是其他人编写的可以配置的机器人

尝试过程中遇到了上文的*枚举数据设置*问题，最初不知是哪出了问题
实在没有办法试了试这个，这个编译起来也很麻烦，不过大致步骤和其他编写了cmake文件的程序差不多。

记录一下编译步骤

1. 同样先`git clone`下载源码

2. 然后需先要编译好sc2client-api静态库，然后将`s2client-api/build/generated`,`s2client-api/include`,`s2client-api/contrib/protobuf/src`三个路径下的头文件放到mingw64的安装目录下的include文件夹下，具体结构参考[CommandCenter](https://github.com/davechurchill/CommandCenter)的linux下的编译方法说明部分。这样做是为了configure时能找到头文件。

4. 源码中cmake文件的库引用路径只适用于Linux，因此需要修改库的引用路径。在cmake文件中，找到commandCenter/cmake/FindSC2Api.cmake，将54行左右修改成如下(修改了PATHS部分的静态库搜索路径，并且在foreach循环后面添加了一个ws2_32库，civetweb需要)

```cmake
# Search for SC2Api libraries.
foreach(COMPONENT sc2api sc2lib sc2utils sc2protocol civetweb protobuf ws2_32)
    string(TOUPPER ${COMPONENT} UPPERCOMPONENT)

    find_library(SC2Api_${UPPERCOMPONENT}_LIB
        NAMES
            "${COMPONENT}"
        PATHS
            "D:/environment/mingw64/lib"
            "D:/environment/mingw64/lib"
        PATH_SUFFIXES
            "sc2api"
        NO_DEFAULT_PATH
    )

    if(SC2Api_${UPPERCOMPONENT}_LIB)
        mark_as_advanced(SC2Api_${UPPERCOMPONENT}_LIB)
        list(APPEND SC2Api_LIBRARIES "${SC2Api_${UPPERCOMPONENT}_LIB}")
    else()
        message(STATUS ${COMPONENT} " not found!")
        set(SC2Api_FOUND FALSE)
    endif()
endforeach()
```

5. 然后就可以新建一个build目录在其下通过cmake-gui生成makefile文件后，执行mingw32-make

6. 最后在build/bin下找到可执行文件，执行（执行时所在目录下需要有BotConfig.txt）
