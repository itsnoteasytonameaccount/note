---
title: zigbee接口使用问题
tags:
- 物联网
- zigbee
- uart
- 串口
- usb
date: 2018/11/30
---

# 前言

zigbee模块要用uart口，但不熟悉，学习了一下，做个记录，顺带学了其他端口

[参考了此处](https://blog.csdn.net/zhuyongxin_6688/article/details/78001767)

<!-- more -->

# uart串口

对应uart总线
嵌入式串口常指uart

接口线为：
    vcc直流正电极
    Rt输出
    RX接收
    gnd接地

TTL电平

# com串口

老pc机连接显示器用的串口

9个针脚

rs232电平

# spi接口

对应spi总线
至少占用4引脚,时钟,使能,输入，输出

# usb

4线:
    红色火线输出直流正电压
    绿色data+接发数据
    白色data-接发数据
    黑色负极
