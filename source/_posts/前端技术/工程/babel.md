---
title: babel学习笔记
date: 2020/01/05
tags:
  - babel
---

# 概述

[官方文档](https://www.babeljs.cn/docs/)
之前没系统地学习过，现在补回，本笔记基于的版本是 babel7，重点为了在 webpack 中使用，故而略去文档中涉及其他打包工具的部分，以及一些命令行使用方法的部分

<!-- more -->

## 概念

据官方文档，babel 是一个 js 编译器，将较新的语言规范编译为向后兼容的语言规范，一边运行在旧版浏览器。
什么都不配置的情况下，babel 做的事近乎于将代码复制一遍，用户根据自己需要兼容什么语法来选用插件(plugin)，当然也可以用别人配置好的插件组合(preset)

## 向导

### 基本用法

以下是不配合框架或打包工具的基本用法

1. 安装核心库、命令行工具、预设、垫片等\*\*

```shell
npm install -D @babel/core @babel/cli @babel/preset-env
npm install -s @babel/polyfill
```

2. 编写配置文件
   后文会说明
3. 运行命令编译和输出

```shell
npx babel src --out-dir lib --presets=@babel/env
```

### 浏览器中

```html
<div id="output"></div>
<!-- Load Babel -->
<!-- v6 <script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script> -->
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
<!-- Your custom script here -->
<script type="text/babel">
  const getMessage = () => "Hello World";
  document.getElementById("output").innerHTML = getMessage();
</script>
```

[具体见此](https://www.babeljs.cn/setup#installation)

### @babel/core

babel 核心功能所在，安装后可直接在代码中引入使用

```js
const babel = require("@babel/core");

babel.transform("code", optionsObject);
```

### 命令行工具

@babel/cli，安装后可在命令行使用

```shell
# 查看帮助
npx babel --help
```

### 插件和预设(plugins & presets)

插件指的是一个个按一定规则转换代码的 js 程序，每个插件都是一个 npm 包，
预设是一组预定义好的插件组合，减少选用插件的工作，常用的如@babel/preset-env

### 配置

分为全局(babel.config.json)和相对(.babelrc.json)的配置,也可以直接写在 package.json(全局)中，也可以写成 js 文件作为模块导出

### polyfill

7.4 以上被废弃，直接引入时，改为引入

```js
import "core-js/stable";
import "regenerator-runtime/runtime";
```

包括了 corejs 和 regenrator runtime 来模拟 es6 环境。
提供可以使用 es6 的新内建对象、静态方法、实例方法的环境，另外还有 regenrator runtime 中实现的 generator 函数。

Polyfill 会污染全局作用域，导致打包结果过大，可以改用 transform-runtime 来避免

甚至如果你知道你确切地需要什么功能，你可以直接从 core-js 中引入

另外如果配置了@babel/env 预设，可以为其添加配置选项 useBuiltIns,并指定 corejs 版本,babel 会自动 require 代码中用到的 es6 api 的垫片而不需要在代码开头引入 polyfill

# 配置方式

全局配置可以是 json 格式，也可以用 js 导出模块或函数。
json 格式时可以写在 package.json 中

```json
{
  "babel": {
    "preset": [],
    "plugins": []
  }
}
```

也可以在需要用到配置文件的项目根目录创建名为 babel.config.json 的文件

```json
{
  "presets": [],
  "plugins": {}
}
```

相对文件的配置.babelrc.json 内容与 babel.config.json 相同，但是可以位于**同一包(package)**（官方文档对使用 monorepo 的特别强调了这一点，因为 Monorepo 一个工程有多个项目(project)）的任何位置，也就是可以对每个子文件夹单独配置，也可以写成 js 文件作为模块导出。

> 注意：默认情况下，局部配置文件之间会覆盖而不是合并，若在 src 中配置了`.babelrc`或`package.json`又在`src/utils`配置了`.babelrc\[.js|.json| \]`，src 中的配置都会失效

通过 js 文件导出配置时，导出的可以是返回配置对象的函数，函数会有一个入参，具体见[官方文档](https://www.babeljs.cn/docs/config-files#supported-file-extensions)

### 命令行工具中配置

```shell
# 启用一个plugin
babel --plugins @babel/plugin-transform-arrow-functions script.js
```

### 直接引入时配置

```shell
require("@babel/core").transform("code", {
  plugins: ["@babel/plugin-transform-arrow-functions"]
});
```

# 命令行使用

[官方文档](https://www.babeljs.cn/docs/babel-cli)

摘几个常用的

```shell
# 编译文件
# 全局安装
babel script.js
# 当前项目
npx babel script.js
# 监听变化--watch,-w，重命名输出文件--out-file,-o
npx babel script.js --watch --out-file script-compiled.js
# 编译目录,--out-dir或-d输出目录
babel src --out-dir output
```

# 插件和预设

### 插件配置

不传参

```json
{
  "plugins": ["pluginA", ["pluginA"], ["pluginA", {}]]
}
```

传带名字的参数

```json
{
  "plugins": [
    [
      "transform-async-to-module-method",
      {
        "module": "bluebird",
        "method": "coroutine"
      }
    ]
  ]
}
```

**预设配置模式与之相同**

### @babel/preset-env

#### browserlist

这是一个在很多前端工具中都有的配置，可通过`@babel/preset-env`的`targets`或`targets.browsers`配置，不过 babel 官网建议配置在一个指定的`.browserlistrc`文件中，因为这样可以与其他工具共享配置，[browerserlist 配置规则见此处](https://github.com/browserslist/browserslist)

如果没设置`targets`,preset-env 默认会查找项目根目录下的`.browerserlistrc`文件

#### targets

指定编译目标的环境等，类型可以是`string`,`Array<string>`,`Object`.

格式为字符串时为上文提到的`browserlist`的格式只不过用逗号分隔行

```json
{
  "targets": "> 0.25%, not dead"
}
```

对象格式的配置方式

```json
{
  "targets": {
    "chrome": "58",
    "ie": "11"
  }
  // 可选chrome, opera, edge, firefox, safari, ie, ios, android, node, electron
}
```

#### useBuiltIns

babel 默认只转化语法（如类的声明），不转化类似 Set,Promise,Map,Proxy,Reflect,Symbol...一些内置的新对象和方法，要使用新的 api，就得用 polyfill。

决定`preset-env`如何处理 polyfills(垫片，模拟实现某些新 api 的代码)
配置这个选项后需要配置 corejs 选项

| 值        | 作用                                                                                   |
| :-------- | :------------------------------------------------------------------------------------- |
| `false`   | 不引入 polyfill                                                                        |
| `'entry'` | 效果和不引入 polyfill 差不多，由用户在 webpack 的 entry 自行引入,或者 html head 中引入 |
| `'usage'` | babel 自动按需引入 polyfill                                                            |

#### corejs

指定 corejs 版本号，可以是 2,3 不配时默认使用 2

### @babel/polyfill

已被废弃，官方建议直接从`core-js`引入

```js
import "core-js/shim"; // included < Stage 4 proposals
import "regenerator-runtime/runtime";
```

由于 babel 默认

**webpack 中**

polyfill 整个引入会导致打包后的代码过大，可参照上文提到的 useBuiltIns 配置来按需引入。
useBuiltIns 不同的配置效果如下。
useBuiltIns 为'usage'时，不用在代码中引入 polyfill
useBuiltIns 为'entry'时，需要在入口配置文件的最开头引入
useBuiltIns 为假值时，需要在代码中引入 polyfill

**其他非浏览器环境**

useBuiltIns 不同的配置效果如下。
useBuiltIns 为'usage'时，不用在代码中引入 polyfill
useBuiltIns 为'entry'时，不知
useBuiltIns 为假值时，需要在代码中引入 polyfill

**浏览器**
需要直接在 node_modules 找到 polyfill.js 用 script 标签引入

### @babel/plugin-transform-runtime

大部分情况下可以用来代替 polyfill，用到一些 api 时无法代替

**与 polyfill 的异同**

polyfill 提供了可以使用**es6 内建对象和方法**以及**原有内建对象原型链上新的方法**(如 Array.prototype.includes)的环境，transform-runtime 创建了可以使用**es6 内置对象和方法**的沙盒环境，polyfill 会污染全局作用域，而 transform-runtime 不会。

也就是说用到`Array.prototype.includes`之类的 api 只能引入 polyfill

transform-runtime 同时能有效减少代码体积：babel 在转化 es6 语法时，会生成一些 helpers 函数，示例如下

```js
class Circle {}

// 转化为
function _classCallCheck(instance, Constructor) {
  //...
}

var Circle = function Circle() {
  _classCallCheck(this, Circle);
};
```

如果每个文件前重复加入这些 helpers 代码会导致代码体积太大，transform-runtime 可以将其改为对特定模块的引入

```js
var _classCallCheck = require("@babel/runtime/helpers/classCallCheck");

var Circle = function Circle() {
  _classCallCheck(this, Circle);
};
```

#### 选项

[其余见官方文档](https://www.babeljs.cn/docs/babel-plugin-transform-runtime)

**corejs**

决定从哪个 runtime 版本引入 helpers

| 值    | 依赖                   |
| :---- | :--------------------- |
| false | @babel/runtime         |
| 2     | @babel/runtime-corejs2 |
| 3     | @babel/runtime-corejs3 |

**helpers**

是否将 helpers 代码转换为对 helpers 的 require

**regenerator**

generator 函数是否使用不污染全局作用域的 regenerator runtime


### transform-runtime,preset-env 和 polyfill 三者区别示例

以代码为例代码如下

```js
class ClassName {
  constructor() {
    let proxy = new Proxy({}, {});
    let request = new Promise(() => {});
    let assign = Object.assign({});
    let isInclude = [1, 2, 3].includes(1);
    let obj = {
      get name(){
        return undefined;
      }
    }
  }
}
```

第一次配置为空

```json
{}
```
结果相当于什么都没干
```js
// 语法
class ClassName {
  constructor() {
    // 完全没法处理
    let proxy = new Proxy({}, {});
    // 可以处理的
    let request = new Promise(() => {});
    let assign = Object.assign({});
    // tranform-runtime没法处理
    let isInclude = [1, 2, 3].includes(1);
    // getter语法
    let obj = {
      get name() {
        return undefined;
      }

    };
  }

}
```

第二次配置`preset-env`决定转化到哪些目标环境，这里选择ie9,ie9以下babel就管不了了，因为babel没办法处理getter和setter语法，ie9以下不支持getter和setter

```json
{
  "presets": [
    [
      "@babel/preset-env",
      // 下面不设也可，默认是es6转es5
      {
        "targets": {
          "ie": 9
        }
      }
    ]
  ]
}
```

可以看到只转化了语法

```js
"use strict";

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

var ClassName = function ClassName() {
  _classCallCheck(this, ClassName);

  var proxy = new Proxy({}, {});
  var request = new Promise(function () {});
  var assign = Object.assign({});
  var isInclude = [1, 2, 3].includes(1);
  var obj = {
    get name() {
      return undefined;
    }

  };
};
```

第三次通过`useBuiltIns: "usage"`引入polyfill

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {
          "ie": 9
        },
        "useBuiltIns": "usage",
        "corejs": 3
      }
    ]
  ]
}
```

可以看到引入了includes,Object.assign,Promise的polyfill，Proxy由于无法用es5实现所以无法处理

```js
"use strict";

require("core-js/modules/es.array.includes");

require("core-js/modules/es.object.assign");

require("core-js/modules/es.object.to-string");

require("core-js/modules/es.promise");

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

var ClassName = function ClassName() {
  _classCallCheck(this, ClassName);

  var proxy = new Proxy({}, {});
  var request = new Promise(function () {});
  var assign = Object.assign({});
  var isInclude = [1, 2, 3].includes(1);
  var obj = {
    get name() {
      return undefined;
    }

  };
};
```

第四次改为使用transform-runtime会如何呢

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {
          "ie": 9
        },
        "useBuiltIns": "usage",
        "corejs": 3
      }
    ]
  ],
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        // 配置为2的原因是corejs3实现了includes，就没办法看出区别了
        "corejs": 2
      }
    ]
  ]
}
```

得到的结果如下，可以看到transform-runtime实现的api全部改名，不会污染全局作用域(这样的好处是对于js库而言的，别人在es6环境用你的东西时不会导致自己环境下的Promise等等全被你替换成了你自己引入的)。
你还能发现用了tranform-runtime的代码没法在浏览器下直接跑。

```js
"use strict";

var _interopRequireDefault = require("@babel/runtime-corejs2/helpers/interopRequireDefault");

// 因为没实现includes,只能用polyfill
require("core-js/modules/es.array.includes");

var _assign = _interopRequireDefault(require("@babel/runtime-corejs2/core-js/object/assign"));

var _promise = _interopRequireDefault(require("@babel/runtime-corejs2/core-js/promise"));

var _classCallCheck2 = _interopRequireDefault(require("@babel/runtime-corejs2/helpers/classCallCheck"));

var ClassName = function ClassName() {
  (0, _classCallCheck2.default)(this, ClassName);
  var proxy = new Proxy({}, {});
  var request = new _promise.default(function () {});
  var assign = (0, _assign.default)({});
  var isInclude = [1, 2, 3].includes(1);
  var obj = {
    get name() {
      return undefined;
    }

  };
};
```

#### 总结

1. babel转化语法
2. polyfill用代码预先实现api以提供es6环境下的内置对象、变量等
3. transform-runtime和polyfill目的相同，但是transform-runtime不污染全局环境，并且transform-runtime对es6环境的模拟不如polyfill完整

# 配置选项

[官方文档](https://www.babeljs.cn/docs/options)

# 升级到 babel7

babel6 到 babel7 是一次破坏性升级，有很多东西不一致，鉴于网上很多博客是 babel6 以前的，因此看技术博客前对 6 到 7 的变化有一定的了解是很有必要的，避免采用了错误的配置导出不符合预期，浪费时间

1. 配置搜索方式的改变
   版本 7 中当遇到子文件夹的 package.json 文件时会停止继续向下搜索，也就是只会搜索单个项目包(package)，并为 monorepo 提供了 babel.config.js 文件来将所有包中的配置集中到一处，当然也可为各个包单独配置，7.1 中还加入了 rootMode 选项来进一步查找。
2. 年度的预设包被废弃
   babel-preset-es20\*全部用 preset-env 代替
3. 阶段(stage)预设被废弃
4. 提案垫片(proposal polyfill)从@babel/polyfill 中废弃，要使用提案需要从 core-js 引入

```js
import "core-js/shim"; // included < Stage 4 proposals
import "regenerator-runtime/runtime";
```

或者从 corejs 单独引入你需要的垫片，列表见[官方文档](https://www.babeljs.cn/docs/v7-migration) 5. 一些包被重命名
babylon 重命名为@babel/parser，仍可使用缩写，如`@babel/preset-react`缩写为`@babel/react`，但官方不建议缩写并考虑移除。 6. 多数包变为了 scoped 包，`babel-`变成了`@babel/` 7. TC39 的提案都带上了`-proposal-`以表明这是提案。
例如：

> `@babel/plugin-transform-function-bind`改为`@babel/plugin-proposal-function-bind`。

> 并且提案被确定部署（即到达 stage-4）时这些包又需要被改名。 6. 移除了一些包里的特性提出年份，因为这没有必要

> `@babel/plugin-transform-es2015-classes`变为`@babel/plugin-transform-classes`。 7. babel6 将文件作用域下的`this`转化为`undefined`并且在模块顶部插入`"use strict"`,babel7 中仅当文件内部有 es6 的导入导出时才有这种表现，如果需要所有文件自动插入`"use strict"`，可使用`transform-strict-mode`插件。 8. 处理 flow 类型注释的预设从 preset-react 中分离出来，现在需要单独引入 9. 配置文件不再允许使用逗号分隔的字符串作为 presets 和 plugins 的值，必须使用数组
