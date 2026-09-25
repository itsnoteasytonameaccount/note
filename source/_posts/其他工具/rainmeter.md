---
title: rainmeter美化工具使用
tags:
- rainmeter
date: 2018/05/31
---
# 前言
rainmeter本来是一款用于监测电脑状况的工具，但由于器强大的扩展功能，被许多人用于美化桌面
<!--more-->

# 开始

## 官方文档
[这里是官方文档](https://docs.rainmeter.net/manual/),养成先看官方文档，官方文档没有帮助再看他人经验的好习惯

## 起步
rainmeter是一款软件，运行时会在你的桌面显示监控你电脑状况的挂件，并且可以通过编辑.ini文件进行挂件及挂件皮肤的自定义，并且挂件可以任意载入卸载，因而可以十分方便的用来美化你的桌面，rainmeter的.ini配置文件可以做到的事有很多。
可以调起可执行程序，播放音频文件，打开文件管路器，调起网页等。另外还可以通过载入扩展文件获得更多功能，可以调用Lua脚本等，可以说是既美观又便捷，同时还轻便

## 编写自定义皮肤
其他不多介绍直接从编写皮肤开始

### section
rainmeter标签分为六种variables,Rainmeter,measure,meter,metadata,style

1.  Rainmeter标签名是固定的，只能有一个，用于声明皮肤的一些整体属性，如更新时间间隔，update等
    ```lua
    ;例
    [Rainmeter]
    Update=1000
    ;1秒更新一次
    ```
2.  Metadata标签名也是固定的，用于声明皮肤的作者，名称，版本,描述，证书信息
    ```lua
    ;例
    [Metadata]
    Author=me
    Version=1.0.0
    Name=skin
    License=无
    Information=第一个皮肤
    ```

3.  Variables标签只能有一个，多出的无效，名称固定，用于声明一些变量，声明的变量可以用在其他标签里
    ```lua
    ;例
    [Variables]
    var1=100
    var2=200
    other_var=string
    ```

4.  measure标签的名字可以自定，不唯一，类似于数据对象，用于定义一些特殊的数据，自动更新的数据等
    一般格式如下
    ```lua
    ;例
    [Mymeasure]
    Measure=Calc
    formula=1
    ```
    注：Measure值代表Measure的类型，有plugin,loop(较新版本),Calc等
    formula为Calc类型measure的求值公式，在每次皮肤更新时会根据公式自动更新measure值
    如下例
    ```lua
    ;每隔一个更新周期measure值加一
    [Mymeasure]
    Measure=Calc
    formula=Mymeasure+1
    ```
    其中Mymeasure在第一次更新时在公式中取值为0，该measure会从1开始递增

5.  meter标签名字可以自定，不唯一，类似于视图对象，用于定义皮肤要展示的内容
    meter也有多种种类，如String,Image等
    String类型的meter显示的是文本，可以text来定义文本内容，也可用MeasureName来引用measure标签的文本值作为内容，两种内容声明不互容
    ```lua
    ;从1开始显示一个数值，每个一个更新时间数值加一
    [Mymeasure]
    Measure=Calc
    formula=Mymeasure+1

    [Mymeter]
    Meter=String
    MeasureName=Mymeasure
    ```
    图片示例
    ```lua
    [Mymeter]
    Meter=Image
    ImageName=/path/of/image/myimagefilename
    ```

6.  style标签用于声明meter样式，类似于css文件，名称自定，可不唯一
    ```lua
    ;样式示例
    [mystyle]
    X=100;位置横坐标
    Y=100;位置纵坐标
    W=100;宽
    H=100;高
    FontFace=华文行楷;字体
    fontsize=20;字号
    fontcolor=255,255,255,255;字体颜色及透明度
    AntiAlis=1;1|0是否抗锯齿
    TextAlign=Center;(Center|Left|Right)[Top|Center|Bottom]
    ClipString=1;自动换行
    SolidColor=255,255,255,255;填充色
    SolidColor2=0,0,0,0;若设置则为两填充色渐变
    GradientAngle=45;渐变角度

    [Mymeter]
    Meter=String
    text=mytext
    Hidden=1;也可以直接在meter中声明样式
    MeterStlye=mystyle
    ```
### 常用属性或语句
1.  hidden,updatedivider
    hidden用于meter是否显示，hidden=1时meter不显示。
    updatedivider可用在meter和measure中，updatedivider的值乘以rainmeter标签中的update值，即为当前Meter或measure的更新时间间隔，当updatedivider=-1时当前measure或meter只在皮肤加载时更新
    ```lua
    [MeasureMy]
    Measure=Calc
    formula=MeasureMy+1
    Updatedivider=-1
    ;以上Measure的值永远为1

2.  引用其他的变量，measure
    可以引用变量或measure来为其他属性赋值应用方法为
    [measureName],#variableName#
    rainmeter4.1以后可以这样
    [#variableName],[&measureName],
    若在表达式中引用需要在外面套括号
    例如:
    ```lua
    ;显示一个文本，该文本一直从上往下运动，又回到开头,一直往复
    [variables]
    max=10

    [measure1]
    Measure=Calc
    formula=(measure1+1)%(#max#)

    [meter1]
    Meter=String
    text=action
    Y=[measure1]
    ```
3.  X,Y样式
    X,Y的样式为数字，可加后缀r,当添加后缀r时,坐标为相对前一个meter的，不加后缀r是相对皮肤左上角而言的
    例：若X=1
          Y=1
          此时如皮肤左上角位于桌面坐标为（100，100），则meter再桌面上的坐标为（101，101）
        若X=1r
          Y=1r
          此时若其前一个meter X=100
                              Y=100
          则当前meter实际 X=101
                          Y=101
        若X=1R
          Y=1R
          此时若其前一个meter X=100,W=10
                              Y=100,H=10
          则当前meter实际 X=111
                          Y=111

4.  update时间
    update时间长度会随cpu使用率自动调整，若CPU使用率太高，update时间会自动拉长，因而即使定义了update=1000更新时间间隔也不一定是1s

5.  条件语句
    If(Above/Equal/Under)Value(似乎是)
    If(Above/Eaual/Under)Action(与上面对应)
    用于measure中，根据值触发事件
    ```lua
    ;例
    [measure1]
    Measure=Calc
    formula=measure1+1
    IfAboveValue=10
    IfAboveAction=measure值大于10时执行的动作
    ```

6.  group
    定义组方便进行组操作
    ```lua
    [Meter1]
    Meter=String
    text=1
    group=mygroup;将Meter1归于mygroup
    ```

7.  事件
    常用事件有
    LeftMouseUpAction,LeftMouseDownAction,MouseOverAction等,这些事件都用于meter中
    ```lua
    LeftMouseUpAction=鼠标事件要执行的动作
    ```

8.  bangs
    事件发生时执行的指令
    常用的有，!SetOption,!Update,!UpdateMeterGroup,!UpdateMeasureGroup,!UpdateMeter,!UpdateMeasure
    ```lua
    ;over事件，更新meter标签的设置
    [meter1]
    Meter=Image
    ImageName=1.png
    MouseOverAction=[!SetOption meter1 ImageName "2.png"][!UpdateMeter "meter1"]
    ;此处若不Updatemeter则需要等更新周期到后才看到更改结果
    ```
    注：!Update指令用于立刻更新所有内容，原理是是暂时覆盖rainmeter中的update属性，对设置了updatedivider=-1的meter或measure无效
