---
title: docker desktop遇到的安装问题
tags:
- docker
- 容器
---

# 问题

很长时间没打开的docker-desktop用不了了原因未知，尝试重新安装

重新安装docker-desktop遇到如下问题

1. 安装

在本地装了wsl2的情况下，安装docker desktop，安装后发现无法启动，最后发现似乎只有较新版本启动不了，安装老版本是可以启动的，因此改为安装`4.24.0`启动成功（但是实际上安装的的docker-desktop功能不正常，总是报Unable to calculate image disk size，界面上也不显示任何东西）

2. 修改镜像数据位置

![image-20240728201017254](.\images\image-20240728201017254.png)

修改上图中的配置后重新启动docker desktop发现下次启动后一直报docker engine stopped，原因未知。

# 解决方案

1. 卸载docker-desktop后用everything找到所有docker-desktop相关的文件夹，全部删除。
2. 删除所有wsl ubuntu上docker相关文件
3. 尝试重新安装



