---
title: 程序编译
tags:
- c
- makefile
date: 2021/01/28
---

每次都是这样，要用东西，以前用过，但是现在忘了，又得找半天，才想起要记笔记

不做深入，简单了解一下

<!-- more -->

# gcc

编译参数

|参数|作用|
|:-|:-|
|-$o$|输出文件名|
|-$O$|优化选项|
|-$l$|链接库|
|-$L$|链接库位置|
|-$I$|头文件搜索路径，可以重复设置多个|
|-$g$|生成调试信息|
|-$ggdb$|尽量生成gdb可用的调试信息|
|-$Wall$|生成所有警告|
|-$w$|不生成警告|
|-$share$|尽量使用动态库|
|-$static$|不使用动态库|
|-$Wl$|发送参数到链接器|
|-$Wa$|发送参数到汇编器|
|-$Wp$|发送参数到预处理器|

## 优化级别

以下内容复制自[GCC 优化级别](https://zhuanlan.zhihu.com/p/196785332)

- -O0不做任何优化，可以用于调试代码
- -O或-O1、-O2、-O3优化程度逐级提高，牺牲了编译时间和可调试性
- -Og在O1的基础上，去掉了那些影响调试的优化，所以如果最终是为了调试程序，可以使用这个参数。
- -Os 是在 -O2 的基础上，去掉了那些会导致最终可执行程序增大的优化，如果想要更小的可执行程序，可选择这个参数。
- -Ofast 是在 -O3 的基础上，添加了一些非常规优化，这些优化是通过打破一些国际标准（比如一些数学函数的实现标准）来实现的，所以一般不推荐使用该参数。
- 如果想知道上面的优化参数具体做了哪些优化，可以使用 gcc -Q --help=optimizers 命令来查询，比如下面是查询 -O3 参数开启了哪些优化

```bash
 gcc -Q --help=optimizers -O3
```

## 编译过程

以c语言为例

+ `gcc -E`: 预处理，处理符号、宏、头文件包含、条件编译等，用到的是预处理器cpp
+ `gcc -S`: 仅编译成汇编文件，不进行汇编或组装，用到的是编译器cc
+ `gcc -c`: 汇编成二进制文件，用到的是汇编器ar
+ `gcc`: 链接成可执行文件，用到的是链接器ld

# 动态库和静态库

动态库和静态库的所处阶段不同，静态库本质上是汇编后的代码，动态库是链接后的代码

## 动态库

不编译入程序，程序运行时加载，多个运行中程序共用同一个动态库时只需在内存中载入一份动态库

编译选项`-shared`

### -fPIC

详见[gcc编译参数-fPIC的一些问题](https://blog.csdn.net/derkampf/article/details/69660050)

> -fPIC 作用于编译阶段，告诉编译器产生与位置无关代码(Position-Independent Code)，则产生的代码中，没有绝对地址，全部使用相对地址，故而代码可以被加载器加载到内存的任意位置，都可以正确的执行。这正是共享库所要求的，共享库被加载时，在内存的位置不是固定的。

> 如果不加-fPIC,则加载.so文件的代码段时,代码段引用的数据对象需要重定位, 重定位会修改代码段的内容,这就造成每个使用这个.so文件代码段的进程在内核里都会生成这个.so文件代码段的copy

### 动态库搜索路径

#### windows

windows下搜索路径:可执行文件目录=>执行目录=>windows目录=>system目录=>path包含的目录
经测试，似乎不可以通过`-Wl,-rpath`指定，也不可通过设置`LD_LIBRARY_PATH`来指定

#### linux

linux下搜索路径：/lib,/usr/lib

可以自行指定，方法如下

__通过设置`LD_LIBRARY_PATH`来指定__

```bash
# 当前路径
export LD_LIBRARY_PATH=.
# 其他路径,多个路径用冒号分隔
export LD_LIBRARY_PATH=/root/mylib:/root/mylib
```

__通过`-Wl,-rpath在编译时指定__

```bash
g++ hello.cpp -L. -lmy -o hello -Wl,-rpath,.
```

## 静态库

编译入程序

编译静态库的方法

```bash
ar -rc xxx.lib xxx1.o xxx2.o
```

# makefile

[make手册(英文)](https://www.gnu.org/software/make/manual/)

## 前置条件和目标等概念

目标指的是编译的目标文件，前置条件指的是编译该目标之前需要编译的目标或者依赖的文件

一般格式

```makefile
目标: 前置条件
    命令列表
```

使用makefile编译时，编译器会自动检查目标文件是否存在、前置条件对应的文件内容相较上一次是否改变，以确定是否需要编译

__伪目标__

类似clean以及其他自定义的标签，不产生真正的同名文件的目标，称为伪目标，若在文件夹存在伪目标的同名文件，会导致伪目标不执行，提示`xxx is up to date`，可以用`.PHONY`告诉编译器哪个是伪目标

例

```makefile
.PHONY: clean
```

## 隐含规则

不提供命令列表的话，会使用隐含规则，推测命令

比如我用mingw测试的隐含规则

```makefile
# 会检查同一目录下的文件
# 若存在main.c，命令是$(CC) -c -o main.o main.c && $(CC) -o main main.o
# 若存在main.o文件，命令是$(CC) main.o -o main
# 若存在main.cpp文件，命令时$(CXX) main.cpp -o main
main:

# 若不存在main.o，命令是$(CXX) main.cpp -o main
# 但若存在main.o，命令会变为$(CC) main.o main.cpp -o main
main: main.cpp

# 其他后缀无隐含规则且不是main这样的特殊目标名只会提示Nothing to do for xxx
anytarget.unknownExt:
```

## 执行编译

用make命令执行，默认执行第一个目标的编译

## 简单语法

__传入参数__

命令行中直接`make PARAM_NAME=PARAM_VALUE`
makefile中使用参数`$(PARAM_NAME)`

__条件__

`ifeq`,`ifneq`,`ifdef`

例

```makefile
ifeq($(CC),gcc)
    libs=$(libs_for_gcc)
else
    libs=$(normal_libs)
endif

bar =
foo = $(bar)
all:
ifdef foo
    @echo yes
else
    @echo  no
endif
```

__示例__

```makefile

FILES_TO_COMPILE = a.o b.o

main: $(FILES_TO_COMPILE) other_cmd
    $(CC) $(FILES_TO_COMPILE) other_file.c -o $(TARGET_NAME) -I $(INCLUDE_PATH) $(LIBS)
# 不常修改的依赖可以单独生成目标，这样下次修改程序代码再次编译时，省去了这些文件的编译时间
a.o: file1.c
	$(CC) -c file1.c
b.o: file2.c
	$(CC) -c file2.c
# 多个目标
c.o d.o: file3.c
    $(CC) -c file3.c
other_cmd:
	# dosomething
clean:
	rm -f a.o b.o
```

__赋值__

```makefile
VARIABLE = value
# makefile开始执行时就扩展。

VARIABLE := value
# 执行到这一句再扩展

VARIABLE ?= value
# 只有在该变量为空时才设置值。

VARIABLE += value
# 将值追加到变量的尾端。
```

`:=`和`=`区别举例

```makefile
A = a
# 相当于'B = '因为makefile执行时A还没有值
B = $(A)
# 相当于'B = a'因为执行到赋值语句时A已经有值
B := $(A)

```

## 通配符

支持`*`任意个任意字符，`?`一个任意字符，`[a-z]`一个字母通配模式

示例

```makefile
clean:
    rm -f *.o
```

## 模式匹配

只有在作为其他目标的依赖的时候才有用，匹配依赖的模式

```makefile

main: f1.o f2.o

# 需要作为依赖
%.o: %.c
# 等同于
f1.o: f1.c
f2.o: f2.c
```

## 自动变量

`$@`指代当前编译目标
`$*`匹配`%`模式匹配到的目标
`$<`匹配第一个前置条件

## 内置变量

`$(CC)`当前c编译器，`$(MAKE)`当前make工具，`$(CXX)`c++编译器

## 路径

makefile中执行的命令环境路径是make程序执行时的路径，而不是makefile所在的路径

```makefile
# parent makefile
# PWD = /root
main: makechild
makechild:
    @$(MAKE) -s -f child/makefile
    # 可以改成
    cd child && @$(MAKE) -s

# child makefile
main:
    @echo $$PWD

# /root下执行make结果是/root而不是/root/child
```

## 函数

wildcard和patsubst需要着重了解一下，非常有用

wildcard用来列举通配符匹配的文件，patsubst用来模式替换

作用举例：无法cd到child文件夹的情况下（比如-I设置了相对路径），在parent文件夹列举child的.cpp文件，并执行命令生成对应的child/.o文件

这个目的在windows下用shell不知道能否达到，但用上述两个函数就很简单

```makefile
CPP_LIST := $(wildcard child/*.cpp)
OBJECT_LIST := $(patsubst %.cpp,%.o,$(CPP_LIST))

main: $(OBJECT_LIST)
    $(CXX) $(OBJECT_LIST) -o main.exe

child/%.o: child/%.cpp
    $(CXX) $< $(INCLUDE_PATH) $(LIBRARY) -o $@
```

## 引入其他文件

相当于用对应文件内容替换include部分

```makefile
include path/to/file
```

## 头文件搜索路径

头文件搜索路径除了编译器默认的路径，以及用户通过-I指定的，还可以用`C_INCLUDE_PATH(gcc)`和`CPLUS_INCLUDE_PATH(g++)`来指定，不同路径间用冒号分隔

## cmd下的for循环在make时有问题

差点气死，同样一条指令`for %l in ('dir /a /b D:\environment\s2client-api\build\bin\*.a') do (ar -x  D:\environment\s2client-api\build\bin/%l)`在cmd下执行就行，在makefile里用mingw32-make执行就报错。
也用`for /?`测试了makefile里确实调的是cmd里的for。
也把下面语句在命令行改了一下`$()`为`%%`执行是正确的，况且把`@`符号去掉后，执行mingw32-make时回显的执行语句明明可以直接在命令行执行，不知道什么毛病。

```makefile
.PHONY: exact

libsc2.dll: exact
	$(CC) -shared *.o *.obj

exact:
	@for /f %l in ('dir /a /b $(LD_LIBRARY_PATH)\*.a') do (ar -x  $(LD_LIBRARY_PATH)/%l)
```

# 参考

1. [GCC 优化级别](https://zhuanlan.zhihu.com/p/196785332)
2. [《GCC编译过程与动态链接库和静态链接库》](https://www.cnblogs.com/king-lps/p/7757919.html) authored by [三年一梦](https://home.cnblogs.com/u/king-lps/)@[博客园](https://www.cnblogs.com/)
3. [《gcc编译参数-fPIC的一些问题》](https://blog.csdn.net/derkampf/article/details/69660050) authored by [逐鹿之城](https://blog.csdn.net/derkampf)@[csdn](https://www.csdn.net/)
4. [《C/C++程序编译过程详解》](https://blog.csdn.net/u012662731/article/details/78520349) authored by [壮二宝](https://blog.csdn.net/u012662731)@[csdn](https://www.csdn.net/)
5. [Makefile ifeq、ifneq、ifdef和ifndef（条件判断）](http://c.biancheng.net/view/7068.html)
6. [《关于DLL搜索路径的顺序问题》](https://blog.csdn.net/pkueecser/article/details/7432168?utm_medium=distribute.pc_relevant_bbs_down.none-task-blog-baidujs-2.nonecase&depth_1-utm_source=distribute.pc_relevant_bbs_down.none-task-blog-baidujs-2.nonecase) authored by [三少GG](https://blog.csdn.net/scut1135)@[csdn](https://www.csdn.net/)