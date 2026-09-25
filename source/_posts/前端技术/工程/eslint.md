---
title: esint、prettier、jest
tags:
  - eslint
  - prettier
  - jest
date: 2020/03/20
---

# eslint

代码规范工具，本文使用版本 6，其他版本可能表现不同

<!-- more -->

## 安装

```bash
npm i -D eslint
```

## 配置

可以在文件中通过配置规则，也可以在配置文件中配置。

### 配置文件应用规则及优先级

1. eslint 配置文件格式可以是`javascript`、`yaml`、`json`、`package.json`,优先级是`.eslintrc.js`>`.eslintrc.yaml`>`.eslintrc.yml`>`.eslintrc.json`>`.eslintrc`>`package.json`，同级配置优先级高的取代优先级低的,可在命令行用`-c`指定额外配置文件。
2. eslint 优先查找需要检测格式的文件的同级配置文件，再逐级向上查找父目录，直到文件系统的根目录。
3. 在层级的配置中与要检测的文件同目录的优先级最高，再逐层往上到根目录，层级的配置中，优先级高的会覆盖优先级低的冲突配置，不同的配置会被合并。
4. 设置可以设置`root:true`，当 eslint 搜索到此配置时不会再向父目录搜索。
5. 行内配置的优先级`/*eslint-disable*/|/*eslint-enable*/`>`/*global*/`>`/*eslint*/`>`/*eslint-env*/`。
6. 命令行配置的优先级`--global`>`--rule`>`--env`>`-c`。
7. 在上述所有位置都没有规则配置时，eslint 会使用用户主目录的`.eslintrc`文件，用户主目录 linux 下是`~/`（如 root 用户为`/root`下），window 下为`%userprofile%`指向的目录。
8. 总优先级**行内配置**>**命令行配置**>**配置文件**>**用户主目录**

### 插件

提供处理器从非 Js 文件中提取 js 代码

```json
{
  "plugins": ["a-plugin"],
  // 格式为"插件名/处理器名"
  "processor": "a-plugin/a-processor"
}
```

### 解析器和环境设置

`parserOptions`配置解析器设置，`parser`指定解析器，`parser`指定别的解析器后，`parserOptions`不一定有效，由解析器决定。

```json
{
  "parserOptions": {
    // 默认为5，可指定6,7,8,9,10
    "ecmaVersion": 6,
    // script或module(ecmascript模块)
    "sourceType": "module",
    // globalReturn允许全局作用域的return语句，impliedStruct全局严格模式，jsx启用jsx语法，experimentalObjectRestScript启用实验性质的object rest/spread properties支持，可同时设置多个
    "ecmaFeatures": { "jsx": true }
  },
  // 新版本更名为@babel/eslint-parser
  "parser": "babel-eslint"
}
```

可通过`env`选项指定环境，[环境列表及各自特点具体见官方文档](https://eslint.bootcss.com/docs/user-guide/configuring)，可通过`globals`选项指定全局变量，以防止调用未声明的变量时报错。

```json
{
  "env": {
    "node": true,
    "es6": true,
    "browser": true,
    // 针对插件启用环境
    "a-plugin/node": true
  },
  "globals": {
    // readonly只读，writable可写，off关闭禁用
    "mycustomVariables": "readonly",
    "Promise": "off"
  }
}
```

### 规则

配置格式`"ruleName": [0, option]`，数组第一项为级别，第二项为规则选项

错误级别

1. `0`或`off`，关闭规则。
2. `1`或`warn`，开启规则，警告。
3. `2`或`error`，开启规则，错误

[建议的规则列表见官方文档](https://eslint.bootcss.com/docs/rules/)

```json
{
  "rules": {
    // 关闭规则
    "eqeqeq": "off",
    "strict": 0,
    "quote": [1, "single"],
    // 插件的规则，格式是"插件名/规则名"
    "a-plugin/ruleName": "warn"
  }
}
```

### 继承规则

从其他地方继承规则

```json
{
  "extends": [
    "eslint:recommended",
    // 从插件继承规则
    "plugin:a-plugin/rulesName"
  ],
  "plugins": ["a-plugin"]
}
```

### overrides

针对单个文件，或匹配到的一组文件覆盖配置。

```json
{
  "plugins": ["a-plugin"],
  "overrides": [
    {
      "files": ["**/*.md"],
      "rules": {
        //   ...
      },
      "processor": "a-plugin/processor",
      "excludeFIles": "*.test.js"
    }
  ]
}
```

### 使用注释配置

可以使用注释注释针对某个特定的文件覆盖`eslint`配置

**设置环境**

```js
/* eslint-env node, mocha */
```

**全局变量**

```js
/* global var1, var2 */
/* global var1:writable, var2:readonly */
```

**规则**

```js
/* eslint eqeqeq: 0, quotes: ["error", "double"] */
```

**禁用和启用 eslint 规则**

```js
/* eslint-disable */
console.log("test");
/* eslint-enable */
// 可以禁用指定的规则
/* eslint-disable no-alert, plugin/rulesName*/
```

**单行禁用**

```js
console.log("test"); // eslint-disable-line
// eslint-disable-next-line
console.log("test");
// eslint-disable-next-line no-console, no-alert
console.log("test");
console.log("test"); /* eslint-disable-line no-console, no-alert */
```

> 注意: 禁用整个文件的规则,eslint 仍会解析代码，故仍需要代码可解析

### 集成到 webpack

[使用 eslint-loader](https://www.npmjs.com/package/eslint-loader)

### 集成到 vscode

[安装 eslint 扩展](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)

集成到 vscode 和集成到 webpack 可以相互配合，`eslint-loader`和 vscode eslint 扩展的配置文件搜索方式是相同的，因此配置好配置后可以相互配合，vscode 中的集成主要用于实时提示 eslint 检测到的问题，webpack 项目中的集成则用于统一项目中的代码规范。

另外可以在 vscode 的配置中设置`codeActionOnSave`如下来启用自动修复格式问题

```json
{
  "editor.codeActionsOnSave": {
    "source.fixAll": true
  }
}
```

# prettier

[prettier 设置](https://prettier.io/docs/en/options.html)

prettier 是常用的格式化工具，如果不和 eslint 配合好，就会带来麻烦，所以这两者需要放在一起学习。
vetur 默认使用的就是 prettier，eslint 有与 prettier 规则不相交的地方，但一般设置`extends: ["eslint:recommended"]`后不会有规则冲突。

一般直接在 vscode 上安装好后，配置默认格式化工具为 prettier 即可，不过使用之后 vscode 上自带格式化工具的一些设置就变得无效了。如`javascript.format`下的配置。

vscode 上 prettier 插件会自动检查当前文件夹及父文件夹的`.editorconfig`和`.prettierrc`文件，若检测到这两个文件，vscode 中的 prettier 配置不是被冲突覆盖，而是全部不再生效，也就是说 setting.json 中所有的 prettier 格式偏好设置全部无效。

# jest

用于单元测试。

安装只需要`npm i jest -g`或者不全局安装`npm i jest -D`

jest 测试的时候因为运行环境是 node，所以可用的引入方式是 commonjs 规范，可以通过 babel 配置 babel 来支持 es6 的导入导出方式，需要配置 babel，不过以下处理会导致目标环境改变，和webpack的babel-loader一起用的时候需要注意

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "target": {
          "node": "current"
        }
      }
    ]
  ]
}
```

[jest 中文文档](https://www.jestjs.cn/docs/getting-started)
