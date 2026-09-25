---
title: 开源邮箱MailSpring安装使用
tags:
  - Visual Studio
date: 2022/10/28 01:11
---

# nylas-mail

由于windows默认邮箱体验太差（看邮件经常内容空白、刷新邮件时非常慢，转半天圈），只能弃用，太难受了，其实它的界面风格我蛮喜欢的。

<!-- more -->

网上找了各种邮件应用最终找到这个开源的[nylas-mail](https://github.com/nylas/nylas-mail)
编写语言里包括js，属于专业对口了，但是npm install了半天发现它依赖的都是些弃用的仓库，有的仓还直接被删了，根本build不了，好在readme最后有指路到mailspring

# mailspring

[mailspring](https://github.com/Foundry376/Mailspring)
可以自己编译，或直接下载release，我这里选择自己编译，主要是为了以后能自定义一些界面

## 编译

`npm run install`再`npm run build`就行

> 如果`install`过程有`node-gyp`的报错，大意是在搜索`Windows SDK`的各个版本但一个都没找到。可以下载一个`Visual Studio Install`，安装时为了避免捆绑安装，选择安装单个组件，找到`Windows SDK`随便安一个版本就行（不能是11的，我安11不被node-gyp识别）

执行完npm命令后，二进制文件就生成了，但点击时会提示缺少[mailsync](https://github.com/Foundry376/Mailspring-Sync/tree/b381582dfc6f744368596310fee53a63f401198c)

# mailsync

[mailsync](https://github.com/Foundry376/Mailspring-Sync/tree/b381582dfc6f744368596310fee53a63f401198c)

这个就非常建议直接下载release了，因为这个工程build时问题层出不穷。
而且也不是UI层而是和系统api的交互应该没有什么自定义的需要，一般情况用作者提供好的就行

## 命令行下尝试编译

### build.cmd

CMakeList.txt不能用，路径都是写死的只能给linux下的cmake用，

只能尝试用build.cmd编译，但是失败，以下内容记录尝试过程，可跳过

build.cmd下的路径也都是写死的，没法直接用，得改改

**而且很可能编不过，本人也没编译成功，不建议用，最好直接用visual studio编译，不过命令行编译过程可以稍微了解以下**

工程默认的build.cmd如下

```bash
# 先运行了一个脚本设置环境
"C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\bin\vcvars32.bat" && ^
# 转到Windows目录
cd Windows && ^
# 执行msbuild
msbuild.exe mailsync.sln /property:Configuration=Release
```

直接执行build.cmd可能会有如下错误

1. 如果vs安装路径是自定的，第一个脚本根本找不到
2. 如果本地msBuild默认platform是x86，会报 MSB4126: 指定的解决方案配置“release|x86”无效
3. 如果上述都通过了，还有可能报一个某某版本的SDK找不到

解决方法，找到第一个脚本的真正路径，并查看`vcvarsall.bat`大致内容如下，以下内容的逻辑是，处理参数，并根据参数设置环境变量，最后根据环境变量生成`__VCVARSALL_VSDEVCMD_ARGS`这一环境变量

```bash
# ...
call :parse_loop
# ...
if "%__VCVARSALL_WINSDK%" NEQ "" (
    set "__VCVARSALL_VSDEVCMD_ARGS=%__VCVARSALL_VSDEVCMD_ARGS% -winsdk=%__VCVARSALL_WINSDK%"
)
# ...
goto :call_vsdevcmd
# ...
:parse_loop
for /F "tokens=1,* delims= " %%a in ("%__VCVARSALL_ARGS_LIST%") do (
    if "%VSCMD_DEBUG%" GEQ "2" (
        @echo [DEBUG:%~nx0] inner argument {%%a}
    )
    call :parse_argument %%a
    set "__VCVARSALL_ARGS_LIST=%%b"
    goto :parse_loop
)

exit /B 0

:parse_argument
# ...
if /I "%1"=="x64" (
    set __VCVARSALL_TARGET_ARCH=x64
    set __VCVARSALL_HOST_ARCH=x64
    set __local_ARG_FOUND=1
)
# ...
set __temp1=%1
if /I "%__temp1:~0,3%"=="10." (
    set "__VCVARSALL_WINSDK=%1"
    echo [%~nx0]: set"__VCVARSALL_WINSDK=%1"
    set __local_ARG_FOUND=1
)
```

可推测出`vcvarsall.bat x64 10.0`即是指定平台`x64`sdk版本`10.0`
但不知为何不生效，仍然报某某版本SDK找不到的错，根据报错找到文件`Microsoft.Cpp.WindowsSDK.targets`

```xml
 <!-- 10.0 -->
    <PropertyGroup Condition="'$(WindowsSDKInstalled)' == '' and '$(WindowsSDK_UAP_Support)' == '' and '$(WindowsSDK_Desktop_Support)' == ''">
      <WindowsSDKInstalled>false</WindowsSDKInstalled>
      <WindowsSDK_UAP_Support>false</WindowsSDK_UAP_Support>
      <WindowsSDK_Desktop_Support>false</WindowsSDK_Desktop_Support>

      <!--此处决定WindowsSDKInstalled是否为true-->
      <WindowsSDKInstalled Condition="Exists('$(WindowsSdkDir)\DesignTime\CommonConfiguration\Neutral\UAP\$(TargetPlatformVersion)\UAP.props')">true</WindowsSDKInstalled>
      <!-- Currently we assume that UWP SDK portion is installed when UAP.props is found-->
      <WindowsSDK_UAP_Support> $(WindowsSDKInstalled)</WindowsSDK_UAP_Support>
      <WindowsSDK_Desktop_Support Condition="Exists('$(WindowsSdkDir)\Include\$(TargetPlatformVersion)\shared\sdkddkver.h') and
                                             Exists('$(WindowsSdkDir)\Lib\$(TargetPlatformVersion)\um\$(PlatformShortName)\gdi32.lib')">true</WindowsSDK_Desktop_Support>
    </PropertyGroup>

    <!-- 产生报错的位置，原因是WindowsSDKInstalled不为true -->
    <VCMessage Code="MSB8036" Type="Error" Arguments="$(TargetPlatformVersion)" Condition="'$(WindowsSDKInstalled)' != 'true'" />
```

最终得到自定义的脚本`custom_build.bat`

```bash
set MODULE_PATH=%cd%
cd "vcvarsall.bat的绝对路径"
call vcvarsall.bat x64 sdk版本号
cd /d %MODULE_PATH%/Windows
call msbuild.exe mailsync.sln -t:Rebuild /property:Configuration=Release /property:TargetPlatformVersion=sdk版本号 /property:DesignTimeBuild=true
cd ..
```

但仍然有错，最后只得换用`visual studio`，捆绑安装一堆东西，使用`visual studio`之后也是困难重重。

## VS编译x86

本人使用的VS版本为VS2019，其他版本表现可能不同

用vs打开工程build即可

### 碰到的问题

build过程中碰到如下报错

1. 找不到某些头文件

   解决方案：关了`visual studio`，工程文件回退一下重新载入

2. C2039 "set_unexpected": 不是 "std" 的成员

   解决方案：[参考此篇](https://cloud.tencent.com/developer/section/1013179)，选择的语法规范版本太高了，需要降至c++17以下（不包含）

3. C2429 语言功能 "if/switch 中的 init-statement" 需要编译器标志 "/std:c++17"

   解决方案：语言规范版本太低

4. C2664 无法将参数 2 从“const wchar_t [5]”转换为“LPCSTR”等等一类报错
   解决方案：可以参考[此篇](https://blog.csdn.net/RYIJU5YUJTY/article/details/125854089)，知道是编码问题，但不一定是改为多字节编码，需要根据报错为置具体问题具体分析，参看报错处的代码

   ```c++
   HCERTSTORE systemStore = CertOpenSystemStore(NULL, L"ROOT");
   
   // wincrypt.h
   WINCRYPT32API
   HCERTSTORE
   WINAPI
   // 第二个参数是LPCSTR类型，报错是“const wchar_t [5]”转换为“LPCSTR”，显然不应该用这个函数
   CertOpenSystemStoreA(
       _In_opt_ HCRYPTPROV_LEGACY      hProv,
       _In_ LPCSTR            szSubsystemProtocol
       );
   WINCRYPT32API
   HCERTSTORE
   WINAPI
   // 第二个参数是LPCWSTR类型
   CertOpenSystemStoreW(
       _In_opt_ HCRYPTPROV_LEGACY      hProv,
       _In_ LPCWSTR            szSubsystemProtocol
       );
   
   #ifdef UNICODE
   // 只能用这个，因此工程编码应当是unicode
   #define CertOpenSystemStore  CertOpenSystemStoreW
   #else
   // 不能用CertOpenSystemStoreA
   #define CertOpenSystemStore  CertOpenSystemStoreA
   #endif // !UNICODE
   ```

5. strncasecmp不接受两个参数，以及附近行的一系列语法错误

   解决方案：查看报错代码
   ```c++    
   if (!has_suffix) {
       if (length - cur_token >= 3) {
           if (strncasecmp(subj + cur_token, "：", 3) == 0) {
               cur_token += 3;
               has_suffix = 1;
           }
       }
   }
   ```
   语法没问题，ide没报错，但是编译的时候识别不出有三个参数，并且附近行一堆语法错误，另外`"："`显然是中文字符，猜测是编码问题，参考[此篇文章](https://blog.csdn.net/qq_42956179/article/details/106075025)，文件另存为GB2312（简中）编码

6. 提示`timespec`重定义

   mailcore引入的一个timespec和系统头文件自带的另一个timespec结构体冲突了，删掉即可、

   ![image-20221108205619668](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221108205619668.png)

至此大功告成，进行一次生成后，之前创建的`custom_build.bat`也能用了

## VS编译x64

编译之前需要修改一下3个module的lib路径，确保不要链接到32位的库

![image-20230227183936603](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20230227183936603.png)

![image-20230227183718634](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20230227183718634.png)

### 碰到的问题

1. 提示`hashmap`或`has_set`被弃用

   ![image-20221109220531932](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221109220531932.png)

   按照报错提示添加宏`_SILENCE_STDEXT_HASH_DEPRECATION_WARNINGS`	

   ![image-20221109220620371](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221109220620371.png)

2. 提示找不到sasl2.lib，mailsync官方未提供64位的lib，github上下一个自行编译，[github地址](https://github.com/cyrusimap/cyrus-sasl/releases)，[官网地址](http://www.cyrusimap.org/sasl/sasl/installation.html)，参考[这篇文章](https://blog.csdn.net/wenjun_xiao/article/details/104455886)，使用vs提供的命令行编译工具`x64 Native Tools Command Prompt for VS 2019`，在源码的lib目录下编译，编译命令`nmake /f NTMakefile STATIC=no`（注：/f指定makefile文件的文件名），编译结果复制到third-party的lib64目录

   ![image-20221109221613897](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221109221613897.png)

3. 找不到libxml2，[此处](https://gitlab.gnome.org/GNOME/libxml2/-/releases)，下载后进win32目录根据readme指示编译，用mingw32-make亲测可行

   示例

   ```bash
   @echo off
   set LIBXML2_PREFIX=D:\projects\git_clone\Mailspring-1.10.5\Mailspring\mailsync\Vendor\mailcore2\Externals
   call cscript configure.js compiler=mingw prefix=%LIBXML2_PREFIX% include=%LIBXML2_PREFIX%\include lib=%LIBXML2_PREFIX%\lib64
   call mingw32-make
   ```

   ![image-20221112195057154](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221112195057154.png)

4. 找不到libctemplate，进[github](https://github.com/OlafvdSpek/ctemplate/releases/tag/ctemplate-2.4)下载源码，vscode编译编译，编译前需要在源码根目录执行如下命令

   ```bash
   python src/htmlparser/generate_fsm.py src/htmlparser/htmlparser_fsm.config > src/htmlparser/htmlparser_fsm.h
   python src/htmlparser/generate_fsm.py src/htmlparser/jsparser_fsm.config > src/htmlparser/jsparser_fsm.h
   python src/htmlparser/generate_fsm.py src/tests/statemachine_test_fsm.config > src/tests/statemachine_test_fsm.h
   ```

   另外记得将默认解决方案改为64位

   ![image-20221113022615869](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113022615869.png)

5. 找不到icudt.lib，进[github](https://github.com/unicode-org/icu/releases/tag/release-70-1), 下载release，然后复制其中的lib64路径下的文件**（注意只能下54版本的，其他不兼容）**

   ![image-20221112233042543](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221112233042543.png)

6. 找不到icule.lib，根据[这篇文章](https://blog.csdn.net/serverxp/article/details/5538878)得知，icule也是icu4c的

   但不知为何直接下载release中编译好的文件里没有对应的lib，只能自己编译，同样进[github](https://github.com/unicode-org/icu/releases/tag/release-70-1), 下载source code**（注意只能下54版本的，其他不兼容）**

   观察工程，目录下只有shell脚本，只能用git-bash之类的类linux环境编译

   如下对应表很关键后面多次需要用到，报哪个外部符号找不到，直接在icu4c的源码中搜索就知道是哪个静态库出问题了

   ![image-20221112223511931](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221112223511931.png)

   **这里有个很坑的点，我发现似乎是安装了wsl之后，windows会在system32路径下放一个bash.exe文件，在这个bash中跑shell，但是你会发现这个bash里的环境变量和windows上的不一样（因为这个bash实际上是wsl虚拟机的环境，git bash上与windows是一样的）。由于我在cmd里直接输入bash，优先选用了System32下的bash，因此跑mingw32-make时提示找不到可执行文件。无奈改用绝对路径执行后可以运行，但仍然会报错cat命令找不到之类的，搞得我一度以为在git bash下使用mingw32-make不可行，但后来[参考这篇](https://my.oschina.net/ctmm/blog/5540160)文章，发现对别人来说是可行的，才恍然大悟**

   也可以通过如下命令，查看其他可选的编译平台及方案等

   ```bash
   ./runConfigureICU -h
   ```

   本人选用的是MinGW方案，步骤如下

   先在源码文件夹下创建文件icu4c_filter.json，内容如下

   ```json
   {
       "strategy": "additive",
       "featureFilters": {
           "normalization": "include"
       }
   }
   ```

   执行如下命令

   ```bash
   # --enable_icuio --enable-static --enable-layout用于生成对应的静态lib
   ./runConfigureICU MinGW --prefix=$PWD/install --enable-rpath --enable-static --enable-icuio --enable-layout --disable-tests --disable-samples --disable-extras ICU_DATA_FILTER_FILE=./icu4c_filter.json
   # 可以不执行install
   mingw32-make -j4 -s
   ```

7. 无法解析的外部符号，根据`_54`后缀猜测是版本之类的问题

   ![image-20221113005724889](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113005724889.png)

   mailsync代码中搜索发现对应符号是unicode的头文件引入的，也就是icu4c

   ![image-20221113010903300](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113010903300.png)

   icu4c源码中查找发现符号在i18n中定义，对应的静态库为icuin.lib

   ![image-20221113011051240](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113011051240.png)

   借助vs19的dumpbin，可以发现确实和版本有关，由于我编译静态库用了icu4c的57版本，编译出的符号带`_57`后缀，而mailsync的某个可能间接依赖了icu4c 54版本，无奈改个版本重新编译

   ```bash
   "D:\environment\Visual Studio\VC\Tools\MSVC\14.29.30133\bin\Hostx64\x64\dumpbin.exe" /SYMBOLS icuin.lib|findstr ucsdet_*
   ```

   ![image-20221113011317457](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113011317457.png)

8. 找不到libcurl_a.lib，[官网](https://curl.se/download.html)下载源码，根据指示用nmake来编译

   ![image-20221113180400794](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113180400794.png)

   打开Developer Command Prompt for VS 2019，进入winbuld目录，执行如下命令即可

   ```bash
   nmake /f Makefile.vc mode=static MACHINE=x64
   ```

   至此不算完，编译时还会报一些无法识别的外部符号，这意味着缺少一些库依赖，需要在工程配置中添加

   ![image-20221113190921097](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113190921097.png)

   比如提示找不到`ldnTounicode`符号，那么就是缺normalliz.lib，找不到

   无法解析的外部符号`__imp_PFXImportCertStore`，那就是缺crypt32.lib

   无法解析的外部符号`__imp__curl_easy_init`，那就是缺Wldap32.Lib

   > 不用担心找不到这些库这些是vs自带的

   ![image-20221113191219247](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113191219247.png)

9. 找不到iconv_a.lib，[镜像地址](https://ftp.gnu.org/pub/gnu/libiconv/)下载手动编译

   同样在git bash环境下用mingw32-make编译，更具install.windows文件夹的指示，咱们的configure命令如下

   ```bash
   # 注意：
   # 1. 由于是在gitbash环境下路径需要是/c/xxxxx/xxxx的格式
   # 2. MAKE必填，你的mingw make命令，不填的话默认命令是make，如果mingw make的文件名不是make.exe，执行会报找不到make
   ./configure --host=x86_64-w64-mingw32 --prefix=你的安装路径 MAKE=mingw32-make --enable-static
   ```

   但这样编译时会报错

   ![image-20221113165441290](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113165441290.png)

   寻找报错原因，查看报错文件源码，发现是有一处宏展开后语法错误了

   ![image-20221113165759298](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113165759298.png)

   再找到对应的Makefile的报错位置，发现这个宏是在DEFS中传入的

   ![image-20221113170108235](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113170108235.png)

   ![image-20221113170140863](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113170140863.png)

   再根据报错内容推测命令传递过程中不知为何被进行了一些加工，将*双引号*去掉了，并且将`/c/xxxx/xxx`格式的路径转化为了`C:\xxxx\xxx`，导致程序中本来预期宏展开后格式为

   ```c
   const char *orig_installdir = "/c/xxxx/xxx";
   ```

   结果现在为

   ```c
   const char *orig_installdir = C:\xxxx\xxx;
   ```

   因此报错

   解决方案1，直接把`INSTALLDIR`这个宏删掉，因为程序中有预处理分支判断，所以不会处理到报错的语句，也就不会报错

   ![image-20221113170859684](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113170859684.png)

   解决方案2，Makefile中再添加一组引号，最终也可编译成功

   ![image-20221113171103481](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221113171103481.png)

10. 找不到libtidy
       [官网下载](http://binaries.html-tidy.org/)，有编译好的版本，把lib目录下的libtyde复制到lib64路径下就行

11. 找不到libsslMD、libcryptoMd、zlib等

    可以直接在libetpan的`third-party/lib64`目录下找到



## 为调试做准备

本来认为不需要这一步的，但后来发现mailSync有个bug，[无法同步163邮箱的邮件](https://community.getmailspring.com/t/cannot-sync-126-163-mails-because-setting-up-snooze-subfolder-in-mail-folders-is-not-allowed/562)。一年多没解决

![image-20221030235325649](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221030235325649.png)

导致我一度想放弃，换一个邮箱应用鼓捣。但苦于没有更好的选择了。其他几个，像是bluemail界面我蛮喜欢的，但有其他bug我也用不了，foxmail界面我又觉的不太喜欢，还会有广告，最后只能尝试把这个问题自行解决看看。

调试前有几个问题需要解决

**注意：mailSync工程有三个模块，故除非特别说明，下文的配置是三个模块同步修改的**

### 问题1：修改优化选项引发的问题

因为编译时的优化可能影响调试，因此把编译选项关了（不太了解只是我这么觉得，好像以前在哪看到过，说不定不用）

![image-20221031000012223](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031000012223.png)

此时执行生成会报告一个错误*“/O2”和“/RTC1”命令行选项不兼容*

这个错误的原因是有两个配置冲突了（见下图），按需要修改其中任意一个即可

![image-20221031000421526](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031000421526.png)

![image-20221031000457354](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031000457354.png)

### 问题2：输出目录找不到pdb文件

首先需要确认如下路径配置没有问题

![image-20221031000923101](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031000923101.png)

其次配置如下参数为是

![image-20221031001033447](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031001033447.png)

另外生成的时候需要选（可能不同版本ide不一样，我不选这个就无法生成pdb，不知道为啥）

![image-20221031001146902](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031001146902.png)

### 问题3：输出文件夹文件不全

最后生成的输出文件夹可能存在不全的情况，只生成了mailcore.dll、libeptan.dll、mailsync.exe，其他依赖的动态库之类的一个都不存在，导致程序无法执行。

比较笨的方法是在配置中找到lib目录，一个个文件复制到输出名录，但效率很低，不可取

![image-20221031003325850](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031003325850.png)

仔细查看工程配置后发下如下脚本配置：

![image-20221031001939602](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031001939602.png)

脚本内容大致如下，用于在生成完成后复制各动态库及结果文件到`../../app/dist`下。

```bash
IF not exist ..\..\app\dist (mkdir ..\..\app\dist) && COPY .\Release\mailsync.exe ..\..\app\dist\ /Y && COPY .\Release\mailsync.pdb ..\..\app\dist\ /Y && ......
# 后面省略
```

但上面脚本有个问题，推测上面那个脚本的本意

`IF not exist ..\..\app\dist (mkdir ..\..\app\dist)`这一句，应当和后面的复制文件的语句是独立的

但是，由于所有命令放在同一行，实际上条件判断后的所有命令都会被包含在`if`分支内（如下图示例），也就导致只要第一个分支不满足，后面所有的复制工作都不会进行

![image-20221031002752048](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031002752048.png)

改正方法则是直接换行，并且dll都用64位代替,改成如下

```bash
IF not exist ..\..\app\dist (mkdir ..\..\app\dist)
COPY .\Release\mailsync.exe ..\..\app\dist\ /Y
COPY .\Release\mailsync.pdb ..\..\app\dist\ /Y
COPY .\Release\libetpan.dll ..\..\app\dist\ /Y
COPY .\Release\mailcore2.dll ..\..\app\dist\ /Y
COPY .\Release\mailcore2.pdb ..\..\app\dist\ /Y
COPY ..\Vendor\libetpan\third-party\bin64\libiconv2.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\bin64\icudt54.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\bin64\icuin54.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\bin64\icuuc54.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\bin64\libsslMD.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\bin64\libcryptoMD.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\lib64\sasl2.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\lib64\libtidy.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\lib64\libxml2.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\lib64\libctemplate.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\lib64\zlib.dll ..\..\app\dist\ /Y
COPY ..\Vendor\mailcore2\Externals\dll\x64\pthreadVC2.dll ..\..\app\dist\ /Y
COPY "C:\Windows\SysWOW64\msvcr100.dll" ..\..\app\dist\ /Y
COPY "C:\Windows\SysWOW64\msvcr120.dll" ..\..\app\dist\ /Y
COPY "C:\Windows\SysWOW64\msvcp100.dll" ..\..\app\dist\ /Y
COPY "C:\Windows\SysWOW64\msvcp120.dll" ..\..\app\dist\ /Y
COPY "C:\Windows\SysWOW64\msvcp140.dll" ..\..\app\dist\ /Y
COPY "C:\Windows\SysWOW64\vcruntime140.dll" ..\..\app\dist\ /Y
XCOPY /rhy "C:\Program Files (x86)\Windows Kits\10\Redist\ucrt\DLLs\x64" ..\..\app\dist\
CD ..\..\app\dist
XCOPY /Y *.* ..\
```



![image-20221031003122469](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031003122469.png)

### 复制文件

最后还有一步，生成的文件被复制到如下目录，需要我们再手动复制一次

![image-20221031003543451](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031003543451.png)

复制到这个目录

![image-20221031003658036](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031003658036.png)

之后就可以启动mailSpring，然后调试了

1. 启动

![image-20221031003845654](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031003845654.png)

2. 调试

![image-20221031003759035](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031003759035.png)

# 成果

最后最可以开心的用上咱们自己编译的mailSpring，可以看到界面还是有些不太美观的地方，以及有些资源化文本还没有翻译，但好在咱们现在可以随便改，随便调，慢慢地改造成最适合自己的

![image-20221031004219704](D:\projects\hexo_blog\source\_posts\mailsync编译\images\image-20221031004219704.png)

# 参考

1. [std::set_unexpected](https://cloud.tencent.com/developer/section/1013179)
2. [error C2664 ”无法将参数 1 从“LPCTSTR”转换为“const char *“ 与指向的类型无关；转换要求 reinterpret_cast、C 样式转换或函数样式转换](https://blog.csdn.net/RYIJU5YUJTY/article/details/125854089)
3. [Visual Studio 2019修改为编码UTF-8的方法（亲测可用）](https://blog.csdn.net/qq_42956179/article/details/106075025)
4. [WIN10 VS2019 编译Cyrus SASL](https://blog.csdn.net/wenjun_xiao/article/details/104455886)
5. [使用 mingw64 在 windows 平台编译 icu4c](https://my.oschina.net/ctmm/blog/5540160)
6. [ICU4C 4.4 静态库的编译](https://blog.csdn.net/serverxp/article/details/5538878)
7. [MSYS2 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/msys2/)