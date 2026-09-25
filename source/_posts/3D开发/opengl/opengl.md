---
title: opengl(1)：库、窗口
tags:
  - opengl
date: 2020/12/18
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站

# 前言

本来想学 webgl，但是把 mdn 上的 webgl 示例学完之后，还是没办法做到绘制想要的图形，因此想看看能不能直接在 opengl 里找到答案

<!--more -->

# 环境

根据参考的文章，选用 [glfw](https://www.glfw.org/),[glad](https://glad.dav1d.de/)这两个库

说明如下

> glfw 是针对 opengl 的 c 语言库,提供了一些渲染物体所需的最低限度的 api<sup>[[4]](#参考)</sup>

> glad 是保存 opengl 函数指针的库<sup>[[4]](#参考)</sup>

也有别的说法

> 另外在这里科普一下glad、glew、glfw、Freeglut的区别：glew（The OpenGL Extension Wrangler Library）是对底层OpenGL接口的封装，可以让你的代码跨平台。glad与glew作用相同，可以看作它的升级版。Freeglut（OpenGL Utility Toolkit）主要用于创建OpenGL上下文、接收一些鼠标键盘事件等等。glfw（Graphics Library Framework）是Freeglut升级版，作用基本一样。通常来说glad和glfw配合使用，比如我上面发的那个网站就是。glew和Freeglut配合使用，比如OpenGL红宝书上面的例子。<sup>[[6]](#参考)</sup>

按照本人使用体验，认为后者更正确

## glad 下载

glad 库需要生成，生成方式有如下几种：

1. 用 python -m glad --generate c --outpath path(前提是已安装 python 的 glad 库)
2. [在线服务](https://glad.dav1d.de/)生成(可能进不去，指定 114.114.114.114 的 dns 服务器就能进去了)
3. 事实上，glfw 里有例子也是需要依赖 glad 的，下载了 glfw 实际上 deps 文件夹里就有 glad

## cmake 编译 glfw 静态库

用 cmake 创建 glfw 的 makefile
cmake 有图形界面 cmake-gui.exe
参考链接中有建立 visual studio makefile 的例子,由于实在不想安装 vs,本文中示例用 mingw.

**操作过程**
需要本地有 Mingw，似乎较老版本的 mingw 不行，我一开始用 codeblocks 自带的 mingw 编译失败了因此重下了一个 mingw64

1. 需要先配置 mingw 的环境变量，否则 Configure 时会无论是否指定编译器位置都会报错
   ![图片无法显示](../image/opengl/Configure失败.png)
   > 配置错环境变量也会报错，比如我环境变量配置的是新下载的 mingw64 的 bin 文件夹位置，但我指定的 mingw 编译器是 codeblocks 自带的 mingw 的，也会报错
   > ![图片无法显示](../image/opengl/配置错误.png)
2. 配置环境变量，源码位置和输出路径后，第一次点击 Configure 指定生成器,配置如图,之后产生一些编译选项供选择。
   ![图片无法显示](../image/opengl/指定生成器.png)
3. 再次点击 Configure 应用选项后,点击 generate 就会生成 makefile 文件
4. 之后 cd 到生成路径下执行 mingw32-make,大功告成
   ![图片无法显示](../image/opengl/大功告成.png)
5. 然后会在 src 路径下产生一个 libglfw3.a 文件，就是所需的静态库，同时 examples 里还会有一些示例程序

# 编译自己的 opengl 程序测试

用的从 [参考链接](#参考)中复制的代码

一开始的编译方式如下，系统是 win10，用的 vscode 编辑代码，用 task.json 配置编译选项如下

```json
{
  "label": "编译opengl文件",
  "command": "g++",
  "group": "build",
  "args": [
    "${env:CUSTOM_LIBS_PATH}/src/glad.c",
    "c/gltest.cpp",
    "-I",
    "${env:CUSTOM_LIBS_PATH}/include",
    "-Llib",
    "-lglfw3",
    "-o",
    "dist/main.exe"
  ],
  "problemMatcher": ["$gcc"]
}
```

用 makefile 的话大致如下

```Makefile
INCLUDE_PATH = $(CUSTOM_LIBS_PATH)/include
REQUIRED_LIBS = -Llib -lglfw3
TARGET = dist/main.exe

main: gltest.o
    $(CXX) gltest.o -o $(TARGET)

gltest.o: glad.o c/gltest.cpp
	$(CXX) -c c/gltest.cpp glad.o $(REQUIRED_LIBS) -I $(INCLUDE_PATH)

glad.o: $(CUSTOM_LIBS_PATH)/src/glad.c
	$(CXX) -c $(CUSTOM_LIBS_PATH)/src/glad.c -I $(INCLUDE_PATH)

clean:
	del glad.o gltest.o dist\main.exe
```

## 出现的问题

1. 头文件顺序错误，需要将 glad.h 放在最前面

   ![图片无法显示](../image/opengl/头文件顺序.png)

2. 缺少依赖，需要引用 gdi32

   ![图片无法显示](../image/opengl/缺少gdi32.png)

用 everything 搜索 gdi32 发现，mingw 路径下有一个 libgdi32，但是链接后还是缺少依赖，最后找到一个系统的 gdi32.dll,编译成功

## 最终改成如下

```Makefile
INCLUDE_PATH = $(CUSTOM_LIBS_PATH)/include
REQUIRED_LIBS = -Llib -lglfw3 -LC:/Windows/WinSxS/amd64_microsoft-windows-gdi32_31bf3856ad364e35_10.0.18362.1_none_2bdc19f7c689fa4b -lgdi32
TARGET = dist/main.exe

main: gltest.o
    $(CXX) gltest.o -o $(TARGET)

gltest.o: glad.o c/gltest.cpp
	$(CXX) -c c/gltest.cpp glad.o $(REQUIRED_LIBS) -I $(INCLUDE_PATH)

glad.o: $(CUSTOM_LIBS_PATH)/src/glad.c
	$(CXX) -c $(CUSTOM_LIBS_PATH)/src/glad.c -I $(INCLUDE_PATH)

clean:
	del glad.o gltest.o dist\main.exe
```

编译，启动，大功告成

![图片无法显示](../image/opengl/启动成功.png)

# hello world

万物之始

创建一个窗口

```c++
#include <glad/glad.h>
#include <GLFW/glfw3.h>

#include <iostream>
void frame_size_callback(GLFWwindow *window, int width, int height);
void processInput(GLFWwindow *window);
void draw(GLFWwindow *window);

int main()
{
    glfwInit();
    // 设置3.3版本
    // opengl主要版本
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    // opengl次要版本
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    // 核心模式
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

    // 没有装饰器，菜单、边框等
    // glfwWindowHint(GLFW_DECORATED, GLFW_FALSE);

    // 透明
    // glfwWindowHint(GLFW_TRANSPARENT_FRAMEBUFFER, GLFW_TRUE);

    // 创建窗口
    GLFWwindow *window = glfwCreateWindow(500, 500, "hello world", NULL, NULL);
    if (window == NULL)
    {
        std::cout << "Failed to create GLFW window" << std::endl;
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(window);
    // 设置回调，每次窗口大小改变，视口大小也改变
    glfwSetFramebufferSizeCallback(window, frame_size_callback);

    // 初始化glad
    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
    {
        std::cout << "Failed to initialize GLAD" << std::endl;
        return -1;
    }

    // 渲染循环
    // 检查是否被要求退出
    while (!glfwWindowShouldClose(window))
    {
        // 轮询键盘输入
        processInput(window);

        draw(window);
        // 交换缓冲区
        glfwSwapBuffers(window);
        // 轮询事件
        glfwPollEvents();
    }

    std::cout << "结束" << std::endl;

    // 退出
    glfwTerminate();

    // 设置视口的大小
    // glViewPort(0, 0, 800, 600);
    return 0;
}

void frame_size_callback(GLFWwindow *window, int width, int height)
{
    // 设置视口位置，前两个参数为左下角坐标，后两个为宽高
	// 参考中的教程说此函数在窗口第一次生成时就会触发，但本人的win10环境中测试，实际上只有窗口大小改变时触发
    glViewport(0, 0, width - 100, height - 100);
}

void processInput(GLFWwindow *window)
{
    // 键是否被按下
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);
}

void draw(GLFWwindow *window)
{
    // 设置清空颜色
    glClearColor(0, 0, 0.4, 0);
    // 使用预设深度和颜色清空
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
}
```

# 示例

用的 opengl3.3,各种操作基本和 webgl2 差不多
核心模式必须使用顶点数组对象，否则图形无法绘制出来

和 webgl 有些区别，主要是着色器

1. webgl 使用 glsl 100，第一行不需要版本声明，webgl2 和 opengl3.3 的着色器代码第一行必须有版本声明
2. webgl 没有 vertex array object
3. glsl 300 es 版本的着色器语言 attribute 改成了 in,varying 变成 out
4. 片段着色器没有了预定义的 gl_FragColor，需要自己声明不以 gl开头的 out 变量

画一个三角形，参考链接给出的示例代码如下

```c++
#include <glad/glad.h>
#include <GLFW/glfw3.h>

#include <iostream>

const char *const vsSource = "#version 330 core\n"
                             "layout (location = 0) in vec3 point;\n"
                             "void main()\n"
                             "{\n"
                             "   gl_Position = vec4(point, 1);\n"
                             "}\n\0";
// const char不能改内容，char * const不能改指向
const char *const fsSource = "#version 330 core\n"
                             "out vec4 color;\n"
                             "void main()\n"
                             "{\n"
                             "   color = vec4(0, 0, 0.5, 0.3);\n"
                             "}\n\0";
// 为了方便用了命名空间，反正自己的代码，也不是在头文件里加，不会出现命名冲突
using namespace std;
void frame_size_callback(GLFWwindow *window, int width, int height);
void processInput(GLFWwindow *window);
void draw(unsigned int program, unsigned int vao);
unsigned int compileProgram(const char *fsSource, const char *vsSource);
unsigned int compileShader(const char *shaderSource, unsigned int type);
unsigned int createBuffer(float data[], int id, int length);
void prepare(unsigned int *buffer, unsigned int *program, unsigned int *vao, int id);

int main()
{
    unsigned int buffer, program, vao;
    glfwInit();
    // 设置3.3版本
    // opengl主要版本
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    // opengl次要版本
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    // 核心模式
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

    // 没有装饰器，菜单、边框等
    // glfwWindowHint(GLFW_DECORATED, GLFW_FALSE);

    // 透明
    glfwWindowHint(GLFW_TRANSPARENT_FRAMEBUFFER, GLFW_TRUE);

    // 创建窗口
    GLFWwindow *window = glfwCreateWindow(500, 500, "hello world", NULL, NULL);
    if (window == NULL)
    {
        cout << "Failed to create GLFW window" << endl;
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(window);
    // 设置回调，每次窗口大小改变，视口大小也改变
    glfwSetFramebufferSizeCallback(window, frame_size_callback);

    // 初始化glad
    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
    {
        cout << "Failed to initialize GLAD" << endl;
        return -1;
    }

    prepare(&buffer, &program, &vao, 1);
    // 渲染循环
    // 检查是否被要求退出
    while (!glfwWindowShouldClose(window))
    {
        // 轮询键盘输入
        processInput(window);

        draw(program, vao);
        // 交换缓冲区
        glfwSwapBuffers(window);
        // 轮询事件
        glfwPollEvents();
    }

    cout << "结束" << endl;

    glDeleteBuffers(1, &buffer);
    glDeleteVertexArrays(1, &vao);
    glDeleteProgram(program);
    // 退出
    glfwTerminate();

    // 设置视口的大小
    // glViewPort(0, 0, 800, 600);
    return 0;
}

void frame_size_callback(GLFWwindow *window, int width, int height)
{
    // 设置视口位置，前两个参数为左下角坐标，后两个为宽高
    glViewport(0, 0, width - 100, height - 100);
}

void processInput(GLFWwindow *window)
{
    // 键是否被按下
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);
}
void prepare(unsigned int *buffer, unsigned int *program, unsigned int *vao, int id)
{
    *program = compileProgram(fsSource, vsSource);
    if (!*program)
    {
        return;
    }
    float data[] = {0, 0.5, 0, -0.5, -0.5, 0, 0.5, -0.5, 0};
    glGenVertexArrays(id, vao);
    glBindVertexArray(*vao);
    *buffer = createBuffer(data, id, 9);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, 0);
    glEnableVertexAttribArray(0);
    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
}

void draw(unsigned int program, unsigned int vao)
{
    // 设置清空颜色
    glClearColor(0, 0, 0, 0);
    // 使用预设深度和颜色清空
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    glUseProgram(program);
    glBindVertexArray(vao);
    // 线框模式
    // glPolygonMode(GL_FRONT_AND_BACK, GL_LINE)
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 3);
}

unsigned int compileProgram(const char *fsSource, const char *vsSource)
{
    unsigned int vertexShader = compileShader(vsSource, GL_VERTEX_SHADER);
    unsigned int fragmentShader = compileShader(fsSource, GL_FRAGMENT_SHADER);
    char log[512];
    if (!fragmentShader || !vertexShader)
    {
        return 0;
    }
    unsigned int program = glCreateProgram();
    int success;
    glAttachShader(program, vertexShader);
    glAttachShader(program, fragmentShader);
    glLinkProgram(program);
    glGetProgramiv(program, GL_LINK_STATUS, &success);
    if (!success)
    {
        glGetProgramInfoLog(program, 512, NULL, log);
        cout << "link program failed:" << log << endl;
        return 0;
    }
    // 以后用不到的shader可以删了
    glDeleteShader(vertexShader);
    glDeleteShader(fragmentShader);
    return program;
}

unsigned int compileShader(const char *shaderSource, unsigned int type)
{
    unsigned int shader = glCreateShader(type);
    int success = 0;
    char log[512];
    // 第二个参数为字符串数量
    glShaderSource(shader, 1, &shaderSource, NULL);
    glCompileShader(shader);
    glGetShaderiv(shader, GL_COMPILE_STATUS, &success);
    if (!success)
    {
        glGetShaderInfoLog(shader, 512, NULL, log);
        cout << "compile shader error:" << log << endl;
        return 0;
    }
    return shader;
}
unsigned int createBuffer(float data[], int id, int length)
{
    unsigned int buffer;
    // id必须唯1
    glGenBuffers(id, &buffer);
    glBindBuffer(GL_ARRAY_BUFFER, buffer);
    // GL_STATIC_DRAW ：数据不会或几乎不会改变。
    // GL_DYNAMIC_DRAW：数据会被改变很多。
    // GL_STREAM_DRAW ：数据每次绘制时都会改变。
    glBufferData(GL_ARRAY_BUFFER, sizeof(float) * length, data, GL_STATIC_DRAW);
    return buffer;
}
```

# 分步骤

为了逻辑清晰，将步骤划分一下

## 初始化 glfw，窗口，glad

```c++

int OpenglWindow::initWindow(int width, int height, const char *title)
{
    glfwInit();
    // 窗口配置
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    // glfwWindowHint(GLFW_DECORATED, GLFW_FALSE);
    glfwWindowHint(GLFW_TRANSPARENT_FRAMEBUFFER, GLFW_TRUE);

    // 创建窗口
    window = glfwCreateWindow(width, height, title, NULL, NULL);
    if (window == NULL)
    {
        cout << "create window failed" << endl;
        glfwTerminate();
        return CREATE_WINDOW_FAILED;
    }
    glfwMakeContextCurrent(window);
    // 大小改变回调
    glfwSetFramebufferSizeCallback(window, setViewport);
    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
    {
        cout << "glad load failed" << endl;
        return GLAD_LOAD_FAILED;
    }
    // 重设视口
    glViewport(0, 0, width, height);
    return 0;
}
```

窗口所有配置参数如下

| 宏                            | 默认值                    | 可取值                                                                               | 作用                                                                                                                                                     | 适用范围                                                                    |
| :---------------------------- | :------------------------ | :----------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| GLFW_RESIZABLE                | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              | 顾名思义                                                                                                                                                 | 窗口模式;GLFW_DECORATED 为 true                                             |
| GLFW_VISIBLE                  | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              | 顾名思义                                                                                                                                                 | 窗口模式;                                                                   |
| GLFW_DECORATED                | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              | 装饰器，关闭放大按钮，边框等是否存在，设为 GLFW_FALSE 时窗口无法被用户改变大小                                                                           | 窗口模式;                                                                   |
| GLFW_FOCUSED                  | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              | 创建时是否有焦点                                                                                                                                         | 窗口模式;创建时不隐藏;                                                      |
| GLFW_AUTO_ICONIFY             | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              | 全屏模式失去焦点时是否自动最小化为图标并且重置之前的 video mode(具体意思不明，详见参考<sup>[[5]](#参考)<sup>)                                            | 全屏模式                                                                    |
| GLFW_FLOATING                 | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              | 总是置顶                                                                                                                                                 | 窗口模式                                                                    |
| GLFW_MAXIMIZED                | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              | 创建时最大化                                                                                                                                             | 窗口模式                                                                    |
| GLFW_CENTER_CURSOR            | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              | 鼠标指针是否跳至新建窗口中央                                                                                                                             | 全屏模式                                                                    |
| GLFW_TRANSPARENT_FRAMEBUFFER  | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              | 透明，这个设为 true 时，在 windows 下，无论颜色透明度设为几，洋红色都是透明的，解决办法查看参考链接<sup>[[5]](#参考)<sup>                                | 无法作用在窗口装饰器上                                                      |
| GLFW_FOCUS_ON_SHOW            | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              | glfwShowWindow 调用时是否获得焦点                                                                                                                        | 未限制                                                                      |
| GLFW_SCALE_TO_MONITOR         | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              | 窗口内容区域大小是否根据显示器内容缩放而调整，窗口初始化时就开始生效                                                                                     | 仅对屏幕像素和坐标映射比例为 1:1 的平台生效，如 windows 和 x11,macos 不生效 |
| GLFW_RED_BITS                 | 8                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       |                                                                                                                                                          |                                                                             |
| GLFW_GREEN_BITS               | 8                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       |                                                                                                                                                          |
| GLFW_BLUE_BITS                | 8                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       |                                                                                                                                                          |
| GLFW_ALPHA_BITS               | 8                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       |                                                                                                                                                          |
| GLFW_DEPTH_BITS               | 24                        | 0 to INT_MAX or GLFW_DONT_CARE                                                       |                                                                                                                                                          |
| GLFW_STENCIL_BITS             | 8                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       |                                                                                                                                                          |
| GLFW_ACCUM_RED_BITS           | 0                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       | 不应当用于新代码                                                                                                                                         |
| GLFW_ACCUM_GREEN_BITS         | 0                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       | 不应当用于新代码                                                                                                                                         |
| GLFW_ACCUM_BLUE_BITS          | 0                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       | 不应当用于新代码                                                                                                                                         |
| GLFW_ACCUM_ALPHA_BITS         | 0                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       | 不应当用于新代码                                                                                                                                         |
| GLFW_AUX_BUFFERS              | 0                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       | 不应当用于新代码                                                                                                                                         |
| GLFW_SAMPLES                  | 0                         | 0 to INT_MAX or GLFW_DONT_CARE                                                       | 多重采样样本数                                                                                                                                           |
| GLFW_REFRESH_RATE             | GLFW_DONT_CARE            | 0 to INT_MAX or GLFW_DONT_CARE                                                       |                                                                                                                                                          |
| GLFW_STEREO                   | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              |                                                                                                                                                          |
| GLFW_SRGB_CAPABLE             | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              |                                                                                                                                                          |
| GLFW_DOUBLEBUFFER             | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              | 双缓冲绘图                                                                                                                                               | 强制为 true                                                                 |
| GLFW_REFRESH_RATE             | 最高可达刷新率            | 0 to INT_MAX or GLFW_DONT_CARE                                                       | 刷新率，GLFW_DONT_CARE 代表采用最高                                                                                                                      | 全屏模式                                                                    |
| GLFW_CLIENT_API               | GLFW_OPENGL_API           | GLFW_OPENGL_API, GLFW_OPENGL_ES_API or GLFW_NO_API                                   | 指定为哪个 api 创建窗口上下文，详见参考<sup>[[5]](#参考)</sup>                                                                                           |
| GLFW_CONTEXT_CREATION_API     | GLFW_NATIVE_CONTEXT_API   | GLFW_NATIVE_CONTEXT_API, GLFW_EGL_CONTEXT_API or GLFW_OSMESA_CONTEXT_API             | 指定用于创建窗口上下文的 api，详见参考<sup>[[5]](#参考)</sup>                                                                                            |
| GLFW_CONTEXT_VERSION_MAJOR    | 1                         | Any valid major version number of the chosen client API                              | 主版本号，若指定不当窗口可能创建失败，详见参考<sup>[[5]](#参考)</sup>                                                                                    |
| GLFW_CONTEXT_VERSION_MINOR    | 0                         | Any valid minor version number of the chosen client API                              | 次版本号，若指定不当窗口可能创建失败，详见参考<sup>[[5]](#参考)</sup>                                                                                    |
| GLFW_CONTEXT_ROBUSTNESS       | GLFW_NO_ROBUSTNESS        | GLFW_NO_ROBUSTNESS, GLFW_NO_RESET_NOTIFICATION or GLFW_LOSE_CONTEXT_ON_RESET         | 指定健壮性策略                                                                                                                                           |
| GLFW_CONTEXT_RELEASE_BEHAVIOR | GLFW_ANY_RELEASE_BEHAVIOR | GLFW_ANY_RELEASE_BEHAVIOR, GLFW_RELEASE_BEHAVIOR_FLUSH or GLFW_RELEASE_BEHAVIOR_NONE | 窗口上下文被释放时的行为，详见参考<sup>[[5]](#参考)和[GL_KHR_context_flush_control](https://www.opengl.org/registry/specs/KHR/context_flush_control.txt) |
| GLFW_OPENGL_FORWARD_COMPAT    | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              | 向前兼容，详见[OpenGL Reference Manual.](https://www.opengl.org/registry/)                                                                               | 不使用 opengles 的情况下                                                    |
| GLFW_OPENGL_DEBUG_CONTEXT     | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              | 是否开启 debug 模式，debug 模式有额外的错误报告和诊断报告,详见[GL_KHR_debug ](https://www.khronos.org/registry/OpenGL/extensions/KHR/KHR_debug.txt)      |
| GLFW_OPENGL_PROFILE           | GLFW_OPENGL_ANY_PROFILE   | GLFW_OPENGL_ANY_PROFILE, GLFW_OPENGL_COMPAT_PROFILE or GLFW_OPENGL_CORE_PROFILE      | opengl 模式，详见[OpenGL Reference Manual.](https://www.opengl.org/registry/)                                                                            |
| GLFW_COCOA_RETINA_FRAMEBUFFER | GLFW_TRUE                 | GLFW_TRUE or GLFW_FALSE                                                              |                                                                                                                                                          | macOs                                                                       |
| GLFW_COCOA_FRAME_NAME         | ""                        | A UTF-8 encoded frame autosave name                                                  |                                                                                                                                                          | macOS                                                                       |
| GLFW_COCOA_GRAPHICS_SWITCHING | GLFW_FALSE                | GLFW_TRUE or GLFW_FALSE                                                              |                                                                                                                                                          | macOs                                                                       |
| GLFW_X11_CLASS_NAME           | ""                        | An ASCII encoded WM_CLASS class name                                                 |                                                                                                                                                          | x11                                                                         |
| GLFW_X11_INSTANCE_NAME        | ""                        | An ASCII encoded WM_CLASS instance name                                              |                                                                                                                                                          | x11                                                                         |

## 编译链接着色器程序

```c++
int OpenglWindow::compileShader(const char *shaderSource, unsigned int type, unsigned int *result)
{
    unsigned int shader = glCreateShader(type);
    int success;
    char log[512];
    glShaderSource(shader, 1, &shaderSource, NULL);
    glCompileShader(shader);
    // 检查是否编译成功
    glGetShaderiv(shader, GL_COMPILE_STATUS, &success);
    if (!success)
    {
        glGetShaderInfoLog(shader, 512, NULL, log);
        cout << "compile shader failed:" << log << endl;
        // 自定义的静态变量
        return COMPILE_SHADER_FAILED;
    }
    *result = shader;
    return 0;
}
int OpenglWindow::linkProgram(const char *fsSource, const char *vsSource)
{
    unsigned int fragmentShader, vertexShader;
    int success;
    char log[512];
    program = glCreateProgram();
    if (compileShader(fsSource, GL_FRAGMENT_SHADER, &fragmentShader))
    {
        return COMPILE_SHADER_FAILED;
    }
    if (compileShader(vsSource, GL_VERTEX_SHADER, &vertexShader))
    {
        return COMPILE_SHADER_FAILED;
    }
    glAttachShader(program, fragmentShader);
    glAttachShader(program, vertexShader);
    glLinkProgram(program);
    // 检查是否链接成功
    glGetProgramiv(program, GL_LINK_STATUS, &success);
    if (!success)
    {
        glGetProgramInfoLog(program, 512, NULL, log);
        cout << "link program failed:" << log << endl;
        return LINK_PROGRAM_FAILED;
    }
    glDeleteShader(fragmentShader);
    glDeleteShader(vertexShader);
    return 0;
}
```

## 创建顶点缓存

> VBO(vertex buffer object) 顶点缓冲对象，存放顶点数据等
> EBO(element buffer object) 索引缓冲对象，存放顶点的索引数据等
> VAO(vertex array object) 顶点数组对象，记录 EBO 和 VBO 的绑定状态

解绑的时候需要注意一下,EBO 不要在 VAO 之前解绑

```c++
// 顶点缓存
void OpenglWindow::createVao(float data[], int length, int id, int location)
{
    unsigned int VBO, VAO;
    glGenVertexArrays(id, &VAO);
    // 生成存
    glGenBuffers(id, &VBO);
    // 绑定缓存为当前缓存
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    // 把数据存入当前缓存
    glBufferData(GL_ARRAY_BUFFER, length, data, GL_STATIC_DRAW);
    // 绑定VAO开始记录状态
    glBindVertexArray(VAO);
    // 设置顶点属性遍历缓存的及方法和下一个函数可以互换
    glVertexAttribPointer(location, 3, GL_FLOAT, GL_FALSE, 0, 0);
    // 启用顶点属性
    glEnableVertexAttribArray(location);
    // 解绑
    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    vao = VAO;
    vbo = VBO;
    id = id;
}

// 加上索引缓冲
void Square::createVao(float data[], unsigned int data2[], int length, int length2, int id, int location)
{
    glGenBuffers(id, &EBO);
    glGenBuffers(id, &vbo);
    glGenVertexArrays(id, &vao);
    glBindVertexArray(vao);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, length, data, GL_STATIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, length2, data2, GL_STATIC_DRAW);
    glVertexAttribPointer(location, 3, GL_FLOAT, GL_FALSE, 0, (void *)0);
    glEnableVertexAttribArray(location);
    glBindVertexArray(0);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    this->id = id;
}
```

## 绘制

以上三步准备就绪后就可以开始绘制

```c++
void Square::draw()
{
    glClearColor(0.4, 0.5, 0.3, 1);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    glUseProgram(program);
    // 绑定VAO相当于绑定了之前准备的顶点缓冲等
    glBindVertexArray(vao);
    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
}
```

## 主循环

需要不断地循环绘制，刷新

```c++
void Square::mainLoop()
{
    while (!glfwWindowShouldClose(window))
    {
        getKeyInput();
        draw();
        glfwSwapBuffers(window);
        glfwPollEvents();
    }
}

```

## main 函数

```c++
int main()
{
    Square square;
    const char *const vsSource = "#version 330 core\n"
                                 "layout (location = 0) in vec3 point;\n"
                                 "void main()\n"
                                 "{\n"
                                 "   gl_Position = vec4(point, 1);\n"
                                 "}\n\0";
    // const char不能改内容，char * const不能改指向
    const char *const fsSource = "#version 330 core\n"
                                 "out vec4 color;\n"
                                 "void main()\n"
                                 "{\n"
                                 "   color = vec4(1, 0.4, 0.5, 1);\n"
                                 "}\n\0";
    float vertices[] = {
        0.5f, 0.5f, 0.0f,   // 右上角
        0.5f, -0.5f, 0.0f,  // 右下角
        -0.5f, -0.5f, 0.0f, // 左下角
        -0.5f, 0.5f, 0.0f   // 左上角
    };
    unsigned int indices[] = {
        // 注意索引从0开始!
        0, 1, 3, // 第一个三角形
        1, 2, 3  // 第二个三角形
    };
    square.initWindow(500, 500, "hello world");
    square.linkProgram(fsSource, vsSource);
    square.createVao(vertices, indices, sizeof(vertices), sizeof(indices), 1, 0);
    square.mainLoop();
    return 0;
}
```

## 几个函数及对象说明

函数的详细文档可以参考[此处](https://docs.microsoft.com/en-us/windows/win32/opengl/gldrawelements)

### glEnable

[参考](https://docs.microsoft.com/en-us/windows/win32/opengl/glenable)

可以用来开启深度测试

```c++
glEnable(GL_DEPTH_TEST);
```

### glShaderSource

`typedef void (APIENTRYP PFNGLSHADERSOURCEPROC)(GLuint shader, GLsizei count, const GLchar *const*string, const GLint *length);`

参数列表

- shader:创建的 shader id
- count:shader 字符串数量
- string:字符串数组
- length:每个字符串的长度,可以是 NULL

### glGetShaderInfoLog

`typedef void (APIENTRYP PFNGLGETSHADERINFOLOGPROC)(GLuint shader, GLsizei bufSize, GLsizei *length, GLchar *infoLog);`

参数列表

- shader:创建的 shader id
- bufSize:infoLog 的长度
- length:用于获取读取到的 log 信息的字符串长度，可以是 NULL
- infoLog:用于获取 log 信息的数组

### glfwSetFramebufferSizeCallback

设置窗口大小改变时的回调函数

**定义**

`GLFWAPI GLFWframebuffersizefun glfwSetFramebufferSizeCallback(GLFWwindow* window, GLFWframebuffersizefun callback);`

这个函数的第二个参数是回调函数指针，无法使用类的成员作为参数（编译会报错，类型转换错误，强转的话编译会有警告但程序能运行）

### glDrawArrays

绘制缓冲数组中存放的顶点

**定义**

`void glDrawArrays(GLenum mode, GLint first, GLsizei count);`

第一个参数是图元类型，有 GL_POINTS, GL_LINE_STRIP, GL_LINE_LOOP, GL_LINES, GL_TRIANGLE_STRIP, GL_TRIANGLE_FAN, GL_TRIANGLES, GL_QUAD_STRIP, GL_QUADS, and GL_POLYGON.
第二个参数是起点

第三个参数是点个数

### glDrawElements

绘制一个元素

**定义**

`void glDrawElements(GLenum mode, GLsizei count, GLenum type, const void *indices);`

- 第一个参数是图元类型，可取:GL_POINTS, GL_LINE_STRIP, GL_LINE_LOOP, GL_LINES, GL_TRIANGLE_STRIP, GL_TRIANGLE_FAN, GL_TRIANGLES, GL_QUAD_STRIP, GL_QUADS, and GL_POLYGON
- 第二个参数是点的个数
- 第三个参数是数据类型，可取 GL_UNSIGNED_BYTE, GL_UNSIGNED_SHORT, or GL_UNSIGNED_INT
- 第四个参数可以是起点位置，也可以是索引数组（不使用索引缓存时）

### glVertexAttribPointer;

用于设置顶点属性指针遍历顶点缓冲的规则

**定义**

`void glVertexAttribPointer(GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const void *pointer);`

- 第一个参数为输入变量的位置，可以直接在着色器代码中定义`location = 数字`,然后此- 处使用，也可以用`glGetAttribLocation`获取
- 第二个参数为每个点的分量个数，可取 2、3、4，即二维还是三维还是四维
- 第三个参数为分量数据类型 GL_SHORT, GL_INT, GL_FLOAT, and GL_DOUBLE.
- 第四个为是否标准化
- 第五个为步长，设为 0 时根据第二、三个参数的值计算（如第二个参数为 3，第三个参数为 GL_FLOAT，结果就是 3\*sizeof(FLOAT)）
- 第六个为首地址偏移量，即从第几个数据开始

### glBufferData

用于往缓冲中填入数据

- 第一个参数为缓冲类型，可以是
  - GL_ARRAY_BUFFER
  - GL_ELEMENT_ARRAY_BUFFER
- 第二个参数为数据长度（字节）
- 第三个参数为数据
- 第四个参数有三种取值
  - GL_STATIC_DRAW ：数据不会或几乎不会改变。
  - GL_DYNAMIC_DRAW：数据会被改变很多。
  - GL_STREAM_DRAW ：数据每次绘制时都会改变。

### glGen\*

提一下，注意第一个参数 id 不能为 0

# 附:imgui 使用

一个较有用的库，可以较为简单地实现绘制一些交互框，方便实时控制绘制过程中的一些参数。(学到模型加载才看到评论区推荐的，可见评论区还是蛮有用的)

在 github 上下载[imgui](https://github.com/ocornut/imgui)源码，然后可以选择直接在工程中引入或者将之编译成动态库。

## 编译成动态库

程序中需要依赖的是 backends 文件夹中和根目录中的文件，根目录所有文件都需要，backends 文件夹根据环境选择需要的文件，比如本文用的是 opengl3 和 glfw，那么需要的文件就是`imgui_impl_opengl3`和`imgui_impl_glfw`

makefile 文件如下

```makefile
# ...

# 列举文件
# CUSTOM_LIBS_PATH为自定义的环境变量
IMGUI_OBJS := $(wildcard $(CUSTOM_LIBS_PATH)/imgui/*.cpp)
IMGUI_OBJS := $(IMGUI_OBJS) $(CUSTOM_LIBS_PATH)/imgui/backends/imgui_impl_glfw.cpp $(CUSTOM_LIBS_PATH)/imgui/backends/imgui_impl_opengl3.cpp

# 头文件列表
INCLUDE_PATH = -I $(CUSTOM_LIBS_PATH)/include -I $(CUSTOM_LIBS_PATH)/imgui -I  $(CUSTOM_LIBS_PATH)/imgui/backends -I c/libs/include

# dist是输出文件夹
# libmyopengl.dll是把glfw，glad等所有基本依赖放在一起输出的动态库
dist/libimgui.dll: $(IMGUI_OBJS) dist/libmyopengl.dll
	$(CXX) -shared -fPIC -O3 $(IMGUI_OBJS) -o dist/libimgui.dll $(INCLUDE_PATH) -Ldist -lmyopengl

```

## 程序中使用

代码示例如下

```c++
/**
 * ...
 * 头文件中需要额外引入
 * #include <imgui.h>
 * #include <imgui_impl_glfw.h>
 * #include <imgui_impl_opengl3.h>
 */

// ...

// 然后需要在glfw，glad等全部初始化后初始化imgui
void initWindow()
{
    // ...
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO &io = ImGui::GetIO();
    ImGui_ImplGlfw_InitForOpenGL(window, true);
    ImGui_ImplOpenGL3_Init("#version 330 core");
}

void draw()
{
    // ...
    
    // 绘制函数添加如下代码即可
    ImGui_ImplOpenGL3_NewFrame();
    ImGui_ImplGlfw_NewFrame();
    ImGui::NewFrame();

    // imgui库自带的绘制示例demo的方法，参数用于控制窗口是否显示
    ImGui::ShowDemoWindow(NULL);

    ImGui::Render();
    ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());
}

// ...
```

# 参考

1. [随笔分类 - OpenGL](https://www.cnblogs.com/zhxmdefj/category/1502112.html) authored by [zhxmdefj](https://www.cnblogs.com/zhxmdefj/)@[博客园](https://www.cnblogs.com/)
2. [C++ / OpenGL | “undefined reference to \`\_imp\_\_ChoosePixelFormat@8\`” and more](https://stackoverflow.com/questions/27753604/c-opengl-undefined-reference-to-imp-choosepixelformat8-and-more)@[stackoverflow](https://stackoverflow.com/)
3. [OpenGL 管线中的深度测试和模板测试为什么在片段着色器之后而非](https://www.zhihu.com/question/40576087/answer/775033162)@[知乎](https://www.zhihu.com/)
4. [LearnOpenGL CN](https://learnopengl-cn.github.io/01%20Getting%20started/03%20Hello%20Window/)
5. [GLFW:window guide](https://www.glfw.org/docs/latest/window.html#window_hints)
6. [醉里挑灯看剑的回答](https://www.zhihu.com/question/264132001/answer/729626917)@[知乎](https://www.zhihu.com)