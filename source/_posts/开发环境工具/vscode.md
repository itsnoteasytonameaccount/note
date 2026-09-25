---
title: vscode使用
date: 2020/07/12
tags:
  - vscode
---

# 前言

工具在手却不会用，或发挥不了其优点是一件很值得惭愧的事.
因此决定看文档学习一下 vscode 的使用方法,弥补一下。

<!-- more -->

先贴上默认快捷键汇总

<img src="../image/vscode/KeyboardReferenceSheet.png">

# 编辑技巧

这真是很值得消化的一部分，能有效提高代码编写效率，惭愧的是我以前只会其中的一小部分

| 快捷键                             | 功能                         |
| :--------------------------------- | :--------------------------- |
| alt+左键                           | 在点击处添加光标             |
| 鼠标中键拖动/ctrl+shift+alt+方向键 | 多行选择                     |
| ctrl+shift+L                       | 选中所有与光标处相同的内容   |
| ctrl+D                             | 添加选中下一处匹配           |
| shift+alt                          | 鼠标拖动选择矩形区域         |
| alt                                | 滚轮快速滚动                 |
| shift+alt+方向键上/下              | 向上或下复制一行             |
| shift+alt+方向键左/右              | 缩小或扩展选中区域           |
| alt+方向键上/下                    | 上/下移一行                  |
| ctrl+shift+o                       | 选中文本中的标记             |
| ctrl+T                             | 选中工作区的标记             |
| ctrl+G                             | 跳至指定行                   |
| ctrl+U                             | 光标回到上一个位置           |
| ctrl+K ctrl+X                      | 消除选中行末空格             |
| ctrl+shift+F                       | 文本格式化                   |
| ctrl+K ctrl+F                      | 格式化当前选中区             |
| ctrl+K ctrl+0/J                    | 折叠或展开所有区域           |
| ctrl+K ctrl+/                      | 折叠所有注释                 |
| ctrl+L                             | 选择当前行                   |
| ctrl+shift+V                       | 打开 markdown 预览           |
| ctrl+K V                           | 分窗口打开 markdown 预览     |
| ctrl+W                             | 关闭当前文件                 |
| ctrl+pagedown/pageup               | 切换到下/上一文件标签        |
| ctrl+shift+K                       | 删除一整行                   |
| ctrl+shift+space                   | 在函数括号处手动触发参数提示 |
| ctrl+i                             | 手动触发代码提示             |

**另外一些文本编辑器中的通用快捷键**

| 按键           | 功能                 |
| :------------- | :------------------- |
| shift+移动光标 | 选中光标移动过的节点 |
| ctrl+Home/end  | 跳至文本首/尾        |
| ctrl+backspace | 退格一个单词         |
| ctrl+delete    | 删除一个单词         |
| home/end       | 移至行首/尾          |

**其他**

可以通过设置 editor.rulers 来自定义竖直标尺距离

```json
{
  "editor.rulers": [20, 40, 60]
}
```

# 界面导航

可以鼠标悬停在对应的位置查看到快捷键

| 快捷键               | 功能                              |
| :------------------- | :-------------------------------- |
| ctrl+,               | 打开设置                          |
| ctrl+shift+M/U/Y     | 底部 debug 控制台等的显示隐藏选项 |
| ctrl+`               | 打开终端                          |
| ctrl+J               | 显示，隐藏底部状态栏              |
| ctrl+shift+E/F/G/D/X | 侧栏的几个选项                    |
| ctrl+K M             | 切换文档语言                      |
| ctrl+K ctrl+T        | 切换主题                          |
| ctrl+K ctrl+S        | 自定键位                          |
| ctrl+B               | 显示，隐藏侧栏                    |
| ctrl+K z             | zen 模式                          |
| ctrl+\\              | 打开新窗口                        |
| ctrl+1/2/3/4         | 在不同窗口间切换                  |
| ctrl+Q               | 切换工具栏                        |
| ctrl+P               | 打开最近文件                      |
| ctrl+O               | 打开文件                          |
| ctrl+K ctrl+O        | 打开文件夹                        |
| ctrl+K W             | 关闭所有标签                      |
| ctrl+K U             | 关闭右侧                          |
| ctrl+R               | 重新载入                          |

**以及一些浏览器常用快捷键也可用在 vscode**

| 快捷键           | 功能         |
| :--------------- | :----------- |
| ctrl+pgdown/pgup | 切换标签     |
| ctrl+w           | 关闭标签     |
| ctrl+tab         | 切换文件标签 |

# 配置和自定义

## .vscode 文件夹

这个文件夹可以存放一些配置文件

## 编辑配置

由于配置太多记录一些编辑器和文件常用配置
[其余见官网](https://code.visualstudio.com/docs/getstarted/settings)
[另 vetur 的配置文档](https://vuejs.github.io/vetur/formatting.html#formatters)

```json
{
  // enter、tab是否接受代码建议
  // - on 是
  // - off 不是
  // - smart enter是tab不是
  "editor.acceptSuggestionOnEnter": "on",
  // 分号
  "javascript.format.semicolons": "remove",
  // 函数括号前加空格
  "javascript.format.insertSpaceBeforeFunctionParenthesis": true,
  // 当词语提交时是否接受代码提示
  // 词语提交指的是变量名或函数名写完后,输入[分号 括号]等分界符
  "editor.acceptSuggestionOnCommitCharacter": false,
  // 保存时的代码动作
  "editor.codeActionOnSave": {},
  // 当文件是单文件时,是否自动检测代码缩进
  "editor.detectIndentation": true,
  // 粘贴,保存,输入时是否自动格式化
  "editor.formatOnSave": true,
  "editor.formatOnPaste": true,
  "editor.formatOnType": true,
  // 自动格式化时的模式
  // - file 整个文件
  // - modifications 修改了的地方(依赖于source control)
  "editor.formatOnSaveMode": true,
  // 按tab时是否自动插入空格
  "editor.insertSpaces": true,
  // ctrl+滚轮是否可以控制缩放
  "editor.mouseWheelZoom": false,
  // 多选操作的控制键
  // - ctrlCMD ctrl
  // - alt alt
  "editor.multiCursorModifier": "alt",
  // 何时显示代码提示
  "editor.quickSuggestions": {
    "other": true,
    "comments": false,
    "strings": false
  },
  // 折叠控制键什么时候显示
  // - mouseover 鼠标悬停时
  // - always 总是
  "editor.showFoldingControls": "mouseover",
  // 建议插入模式，vscode对不同语言的这一设置，设置了不同的默认设置，因此修改需要针对各语言修改
  // - replace 替换后面的单词
  // - insert 插入
  "editor.suggest.insertMode": "insert",
  // 显示代码建议时选择哪个建议
  "editor.suggestSelection": "recentlyUsed",
  // 制表符空格数
  "editor.tabSize": 4,
  // 自动换行控制
  // - off 从不
  // - on 超过窗口宽度
  // - wordWrapColumn 基于wordWrapColumn
  // - bounded 基于最小窗口和wordWrapColumn
  "editor.wordWrap": "off",
  "editor.wordWrapColumn": 80,
  // 关联语言和文件扩展
  "files.associations": {},
  // 文件修改后自动保存
  "files.autoSave": "off",
  // 自动保存延时
  "files.autoSaveDelay": 1000,
  // 默认编码
  "files.encoding": "utf8",
  // 回车
  "files.eol": "auto",
  // 排除文件不在文件导航栏显示
  "files.exclude": {
    "**/.git": true,
    "**/.svn": true,
    "**/.hg": true,
    "**/CVS": true,
    "**/.DS_Store": true
  },
  // 禅模式是否隐藏行号
  "zenMode.hideLineNumbers": true,
  // 禅模式是否隐藏状态栏
  "zenMode.hideStatusBar": false,
  // 一个子文件的文件夹像java一样采用parent/child的模式显示,而不是层级
  "explorer.compactFolders": true,
  // 文件导航栏是否允许拖拽
  "explorer.enableDragAndDrop": true,
  // 全局搜索应排除的路径
  "search.exclude": {
    "**/node_modules": true,
    "**/bower_components": true,
    "**/*.code-search": true
  },
  // html后接新行的标签
  "html.format.extraLiners": "head, body, /html",
  // 属性换行
  "html.format.wrapAttributes": "auto",
  // 格式化时分号处理
  // - ignore 不做任何处理
  // - insert 表达式尾部插入分号
  // - remove 移除不必要的分号
  "javascript.format.semicolons": "ignore",
  // 正确性检查的一些配置
  "css.lint.*": "any",
  "less.lint.*": "any",
  "scss.lint.*": "any",
  // 终端命令行程序
  "terminal.integrated.shell.windows": "C:\\Windows\\System32\\cmd.exe",
  // 环境设置
  "terminal.integrated.env.windows": {}
}
```

## 对各语言进行单独配置

```json
{
  "[javascript]": {
    // ...
  }
}
```

> 注意: vscode 中针对语言修改配置时，带注释的 json 应选语言是 jsonc,例如 jsconfig.json、setting.json 的语言都是 jsonc.

# 全局搜索

**文件匹配规则**

> `*`匹配路径中一个以上字符
> `?`匹配路径中一个字符
> `**`匹配路径中任意个字符
> `{}`满足多种条件其一
> `[]`一个字符的可取值范围

# snippets

提高效率的重要功能
自动完成一些重复的代码块

vs code 内建了一些语言的 snippets,例如:javascript,Typescript,MarkDown,和 PHP

## 自定义自己的 snippets

**file**>**preferences**>**user snippets**中选择特定的语言，可以自定义用户 snippets
或者直接`ctrl+shift+P`输入`user snippets`命令。

可以选择`new global snippets file`创建对所有语言生效的 snippets，
可以选择`new global snippets file for '...'`创建仅对当前项目生效的 snippets
文件存在.vscode 中

**snippets 文件示例如下**

```json
{
  "For Loop": {
    // 触发关键词
    "prefix": ["for", "for-const"],
    // 内容，${}占位符在生成代码块后可以用tab键跳转
    "body": ["for (const ${2:element} of ${1:array}) {", "\t$0", "}"],
    // 描述
    "description": "A for loop."
  }
}
```

## snippets 编写规则

### tab 切换点(tabstops)

按 tab 键会在 snippets 中的`$num`标记位置上切换，顺序从 1-10(0)，如果序号相同会有多个光标，可以同步编辑

### 占位符(placeholders)

tabstops 处默认显示的内容，例如`snippets:${1:first}`,生成的代码块就默认是`snippets:first`，可以嵌套，可以使用 tab 选中占位符

### 选项(choice)

`${1|one,two,three|}`可以在两个`|`符号中包括的几个内容之间进行选择的 tabstops。

### 变量(Variables)

`$变量名`或者`${变量名:default}`
定义好的特定的变量，若变量不存在采用 default 的内容

具体有哪些变量可以看官方文档 [变量定义](https://code.visualstudio.com/docs/editor/userdefinedsnippets#_snippet-syntax)

### 具体语法

以下表格中 `*`代表前一个字符或组可以重复零到多次，`+`代表前一个字符或组可以重复一到多次，`？`代表前一个字符或组可以重复零或一次，`[_0-9a-z]`代表在`_`、`0`到`9`、`a`到`z`之间取值的字符，`.`代表任何字符

| 类型          | 格式                                                                                                                                                                              | 描述                             | 例                                  |
| :------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------- | :---------------------------------- |
| `any`         | `tabstop` 或 `placeholder` 或 `choice` 或 `variable` 或 `text`                                                                                                                    | 任何类型                         | 见下                                |
| `tabstop`     | \$`int` 或 \${`int`} 或 \${`int` `transform`}                                                                                                                                     | tab 切换点                       | `$1` `${1}` `${1/\\\\d/num/g}`      |
| `choice`      | \${`int`&#124;`text`(,`text`)\*&#124;}                                                                                                                                            | 选项                             | `${1&#124;选项1,选项2,选项3&#124;}` |
| `placeholder` | \${`int`:`any`}                                                                                                                                                                   | 占位符                           | \${1:ceshi}                         |
| `variable`    | \${`var`} 或 \$`var` 或 \${`var` `transform`} 或 \${`var`:`any`}                                                                                                                  | 变量                             | `${TM_FILENAME/(.*)/_/}`            |
| `transform`   | /`regex`/(`format`&#124;`text`)+/`options`                                                                                                                                        | 字符串转化规则                   | `/.*/_/`                            |
| `format`      | \$`int` 或 \${`int`} 或 \${`int`:(/upcase&#124;/downcase&#124;/capitallize)} 或 \${`int`:+if 语句} 或 \${`int`:?if 语句:else 语句} 或 \${`int`:-else 语句} 或 \${`int`:else 语句} | 用来替换匹配到的字符串的替换模式 |                                     |
| `text`        | .\*                                                                                                                                                                               | 任何字符串                       |                                     |
| `int`         | \[0-9\]+                                                                                                                                                                          | 任何数字                         |                                     |
| `var`         | \[a-zA-Z\]\[a-zA-Z0-9\]\*                                                                                                                                                         | 任何合规变量名                   |                                     |
| `regex`       |                                                                                                                                                                                   | 任何合 js 规范的正则表达式值     |                                     |
| `options`     |                                                                                                                                                                                   | 任何合 js 规范的正则表达式选项   |                                     |

### 举例

# emmet

提高效率的重要功能
默认对`html`,`haml`,`pug`,`slim`,`jsx`,`xml`,`xsl`,`css`,`scss`,`sass`和`stylus`等类型语言以及上述类型语言的扩展语言有效。

可以通过一定格式的缩写来编写标记语法、样式等。

## 相关配置

[其余配置见官方文档](https://code.visualstudio.com/docs/editor/emmet)

```json
{
  // 是否启用tab快捷键转化emmet缩略语
  "emmet.triggerExpansionOnTab": true,
  // 不显示缩略语预览
  "emmet.showExpansionAbbreviation": "never",
  // 仅在html/css文件中显示emmet提示
  "emmet.showExpandedAbbreviation": "inMarkupAndStylesheetFilesOnly",
  // 设置缩略语预览的位置
  "emmet.showSuggestionAsSnippets": true,
  "editor.snippetSuggestions": "top",
  // 在不同的文件类型中启用缩略语,如下默认js不支持emmet,键名和键值都应当是vscode支持的语言类型
  "emmet.includeLanguages": {
    "javascript": "javascriptreact"
  },
  "emmet.preferences": {
    // 样式的浏览器前缀
    // 这两个属性才加前缀
    "css.webkitProperties": "border-right,animation",
    // 都不加前缀
    "css.mozProperties": "",
    // 采取默认情况
    "css.oProperties": null,
    "css.msProperties": null
  }
}
```

## 过滤器

过滤器在缩略语生成代码前对代码进行进一步处理

### 两种添加方法

配置文件中添加语言对应的 emmet 过滤器

```json
{
  "emmet.syntaxProfiles": {
    "html": {
      "filters": "bem"
    }
  }
}
```

直接在缩略语后通过`|`符号传递给过滤器，例如`div#page>li|c`使用了注释过滤器.

## 自定义 emmet 片段

用户可以自定义 emmet 片段

> 这里的自定义只能自定义 snippets，其他方面如语法的自定义不知道如何配置，本来我以为能实现在 js 能像 html 中一样，写`parent>children`扩展成`{parent:{children:{}}`之类的。

首先需要设置`emmet.extensionsPath`为你放置自定义代码片段的**文件夹路径**。
修改这一配置后据文档中说需要重启 vscode，但我试了不用重启，可能是版本问题，修改后无效应试着重启一下
然后在文件夹中创建`snippets.json`。

> 注意：window 下`emmet.extensionsPath`路径的格式也应当是`/`分隔的**绝对路径**，否则不起作用，至少对我是这样，网上不知道为有些博客明明没用的配置也写出来误人子弟。
> 另外 markdown 似乎无法启用 emmet
> 若 js 之类的语言以如下方式启用了 emmet

```json
{
  "emmet.includeLanguages": {
    "javascript": "javascriptreact"
  }
}
```

仍需修改 html 对应的 snippets 而不是 javascriptreact 的，似乎是因为 javascriptreact 继承自 html emmet 语法

```json
{
  // 无效
  "javascriptreact": {
    "snippets": {
      "test": "doesn't work"
    }
  },
  // 生效
  "html": {
    "snippets": {
      "html_test": "work"
    }
  }
}
```

[具体参见此处](https://github.com/microsoft/vscode/issues/109381)

配置举例如下

**setting.json**

```json
{
  "emmet.extensionsPath": "/D:/project/emmet/"
}
```

**snippets.json**

```json
{
  "html": {
    "snippets": {
      "ull": "<ul><li></li><li></li></ul>"
    }
  }
}
```

[emmet 文档](https://docs.emmet.io/customization/),可以了解怎么编写 emmet 片段
[vscode 文档](https://code.visualstudio.com/docs/editor/emmet)

# jsconfig.json

用来指示当前文件夹为项目根目录
[官方文档](https://code.visualstudio.com/docs/languages/jsconfig)
不配置此文件时，单个文件以及其引用的依赖被视为一个整体，配置后，`include`包含的文件拥有相同的项目上下文，整个项目被视为一个整体，好处在于，代码提示中可以看到同项目其他文件中定义的模块，并且使用其他文件定义的模块时可以自动导入等等。。。

## 配置

可以用来配合 webpack 的路径别名

```json
{
  "compilerOptions": {
    // 设置path必须要设置baseurl,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    // 支持.jsx扩展名的文件
    // 可惜没有支持,vue扩展名的配置
    "jsx": "react",
    // 自动导入时的导入导出方式
    "module": "es6"
  },
  "include": ["src/**/*"]
}
```

# 类型检查

可以把 typescript 的一些类型检查的特点使用在 javascript 中

## 启用

**单文件中使用注释启用**

```js
// 启用
// @ts-check
// 忽略某一部分的错误
// @ts-ignore
// 不检查当前文件
// @ts-nocheck
```

**项目中配置**

可以在`setting.json`中设置`"javascript.implicitProjectConfig.Checjs":true`来启用,也可以在`jsconfig.json`中设置`"Checkjs":true`来控制

## 补充

可能会导致一些原本正确的代码由于不符合 typescript 规范而报错

# 代码导航

| 快捷键    | 描述             |
| :-------- | :--------------- |
| f12       | 转到定义         |
| alt+f12   | 在分窗口打开定义 |
| shift+f12 | 转到引用         |

# debug

## launch.json

`ctrl+shift+D`打开 debug 栏，点击上方设置的图标创建`launch.json`，有两种模式`launch`模式和`attach`模式，分别是直接启动，和绑定已启动程序。

在`launch.json`中,通过右下角的`add configuration`按钮来添加启动设置。

可以在`launch.json`中设置`stopOnEntry: true`,使调试器停止在开头处。

[配置文件属性](https://code.visualstudio.com/docs/editor/debugging#_launchjson-attributes)

## 启动

点击 debug 栏最上方的启动下拉列表中的一项，启动 debug。
也可在配置`launch.json`后`f5`启动

## 断点

在行号左侧可以设置左击断点，右击选择设置断点、条件断点、输出点
断点处右击可以编辑断点，将其设置为输出点、条件断点、或者设置断点命中次数。
也可在代码行按`f9`设置或取消断点。

1. 运行至输出点时会在在 debug 控制台输出该点表达式中的内容。
2. 条件断点只有断点处设置的表达式为真时，才会被命中到
3. 断点命中次数设置为`=n`后，只有断点第 n 次命中才会中断一次，`>n`则是第 n 次命中以后都会中断，依次类推其他表达式的情况

### 断点无效

可能是本地的运行环境和当前版本 vscode 默认的调试环境版本不匹配。

[见此处](https://www.jianshu.com/p/91980de221da)

`nodejs`调试添加`"runtimeVersion": "版本号"`，这一配置需要本地安装了`nvm-windows`，配置完后需要`nvm use 你配置的版本号`（没安装对应版本需要先用 nvm 安装），即可正常使用。

## 调试

`f5`继续或暂停，`ctrl+shift+f5`重启，`shift+f5`终止，`f10`下一步，`f11`跳入，`shift+f11`跳出。

# 任务

[官网地址](https://code.visualstudio.com/docs/editor/tasks)

`ctrl+shift+p`输入 task，选择`configure task`可以配至 task 文件。
`ctrl+shift+b`运行 build task
菜单栏**terminal**>**Run Task...**选择要运行的命令。

## 主要配置说明

| 属性             | 类型                | 作用                                                                                                                                                                                   |
| :--------------- | :------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `label`          | `String`            | 任务标签                                                                                                                                                                               |
| `type`           | `String`            | 任务类型                                                                                                                                                                               |
| `command`        | `String`            | 任务指令                                                                                                                                                                               |
| `windows`        | `Object`            | windows 系统下的替代配置                                                                                                                                                               |
| `group`          | `String`            | 所属的组，建议设为 build，因为只有 build 设置了默认快捷键`ctrl+shift+B`，设为 test 需要定义一下`run test task`的快捷键，设为其他值的话就得输入`alt+T R`打开`run task`在选择自己的 task |
| `presentation`   | `Object`            | 任务输出如何显示                                                                                                                                                                       |
| `options`        | `Object`            | 环境相关(`CWD`,`ENV`,`SHELL`)的附加命令                                                                                                                                                |
| `args`           | `Object&#124;Array` | 命令行参数                                                                                                                                                                             |
| `problemMatcher` | `Array`             | 对任务输出进行预处理使其有高亮显示等                                                                                                                                                   |

官网上说可以按`ctrl+space`打开代码提示查看所欲属性，然而我没修改配置，`ctrl+space`却无效，`ctrl+i`才有效。

对于带空格或特殊符号的单个参数或指令，如带空格的路径等，默认会加上引号，cmd.exe 下是双引号，其他是单引号。

可以通过配置来控制单双引号，

```json
{
  "label": "dir",
  "type": "shell",
  "command": "dir",
  "args": [
    {
      "value": "folder with spaces",
      "quoting": "escape"
    }
  ]
}
```

其中 `quoting`取值为 `escape` `strong` 或 `weak`，escape 是使用转移字符代替空格，strong 就是上文提到的默认行为，weak 都是双引号。

## presentation 可选配置

**reveal**: 是否执行任务时是否将终端打开并置顶。

    . *always* 总是打开并置顶
    . *never* 用户必须手动打开只能高端
    . *silent* 只有报错或警报时打开

**focus** 终端是否获得焦点
**echo** 是否回显任务执行结果

## 可使用的变量

[官网地址]](https://code.visualstudio.com/docs/editor/variables-reference)

常用的一个是环境变量格式是`${env:你要使用的变量名}`

> 注意: 在 options 中定义的环境变量，只会传递到要执行的程序中，无法用于 command,args 等属性上

```json
{
  // ...
  "options": {
    "env": {
      "MY_VARIABLE": "${env:MY_VARIABLE}/value"
    }
  },
  "type": "progress",
  // 以下配置实际结果为/main.exe
  "command": "${env:MY_VARIABLE}/main.exe",
  // 同上
  "command": "%MY_VARIABLE%/main.exe"
  // ...
}
```

> 如下设置则可

```json
{
  // ...
  "options": {
    "env": {
      "MY_VARIABLE": "${env:MY_VARIABLE}/value"
    }
  },
  "type": "shell",
  // 以下配置符合预期
  "command": "%MY_VARIABLE%/main.exe"
  // ...
}
```

因为 type 为 shell 的任务，是通过`cmd /c`执行的，环境变量已被传递到 cmd.exe 中

## 复合指令

任务可以依赖其他任务，被依赖的任务会在依赖他们的任务之前执行，只需要设置 `dependsOn`即可，`dependsOn`为字符串数组，字符串中是其他任务的 label 值。

## 示例：c 语言的编译任务和调试

环境是 windows 首先下载编译工具，mingw64
[下载地址](https://sourceforge.net/projects/mingw-w64/files/mingw-w64/mingw-w64-snapshot/)

选择其中 mingw-w64 gcc 的任意一个版本

[不同版本的区别](https://www.pcyo.cn/linux/20181212/216.html)

接下来安装 vscode 的`c/c++`拓展

下载好后解压，自行配置环境变量，再在 vscode 里如下配置

事实上设置了环境变量`MINGW_HOME`后不做如下配置也可以，经测试，c/c++插件能自动检测这个变量，不过若是在 vscode 启动后修改了环境变量，需要把 vscode 关了重启修改才会生效。

路径分隔符可以是`/`也可以是`\\`

**setting.json**

```json
{
  "C_Cpp.default.includePath": [
    // 头文件默认引入位置，MINGW_HOME是自定义的环境变量
    "${env:MINGW_HOME}/**"
  ]
}
```

可以不配置`c_cpp_properties.json`，采用默认配置，但如若配置了`c_cpp_properties.json`并且配置了`includePath`，需要在`includePath`数组中加入`${default}`代表在 setting.json 中配置的默认头文件引入位置。

然后用复合指令编译生成可执行文件，`ctrl+shift+p`后输入 tasks 选择`configure task`，会生成配置模板。
**tasks.json**

```json
{
  "tasks": [
    {
      "label": "prepare",
      "type": "shell",
      // 生成debug目录
      "command": "if not exist debug (mkdir debug)",
      "presentation": {
        "echo": true,
        "reveal": "silent",
        "focus": false,
        "panel": "shared",
        "showReuseMessage": true,
        "clear": false
      }
    },
    {
      "label": "build test",
      "type": "shell",
      "command": "gcc",
      "args": [
        // 你自己需要编译的文件
        // 此处${file}为vscode替换变量，指代打开的文件
        "${file}",
        "-o",
        // ${fileBasename}当前打开的文件去掉扩展名
        "debug/${fileBasenameNoExtension}.exe",
        // 包含调试信息
        "-g"
      ],
      "group": "build",
      "presentation": {
        "reveal": "silent"
      },
      "problemMatcher": "$msCompile",
      "dependsOn": ["prepare"]
    }
  ]
}
```

配置启动文件，`ctrl+shift+d`打开 debug 界面，左上角的设置按钮可以新建`launch.json`文件。或者`ctrl+shift+p`再输入 launch 可以找到 open launch json 命令。在 launch.json 文件右下角有 add configuration 按钮，点击后选择`（gdb）启动`模板。

修改可执行文件及 gdb 的路径，再添加 prelaunchTask 选项设置为配置好的编译命令即可。

**launch.json**

```json
{
  // Use IntelliSense to learn about possible attributes.
  // Hover to view descriptions of existing attributes.
  // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "name": "(gdb) 启动",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/debug/${fileBasenameNoExtension}.exe",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      // 自己的Mingw路径，两种路径分隔符都可
      "miDebuggerPath": "D:\\environment\\mingw64\\bin\\gdb.exe",
      "setupCommands": [
        {
          "description": "为 gdb 启用整齐打印",
          "text": "-enable-pretty-printing",
          "ignoreFailures": false
        }
      ],
      "preLaunchTask": "build test"
    }
  ]
}
```

> 注意 如果有多个不同的启动配置，记得看 debug 窗口最上面的默认启动配置是不是你需要的那个，我因为忘了看发现不符合预期浪费了蛮多时间。
