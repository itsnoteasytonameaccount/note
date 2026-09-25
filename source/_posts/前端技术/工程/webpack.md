---
title: webpack的使用
tags:
  - webpack
  - nodejs
  - javascript
date: 2018/05/19
---

# 前言

先看[官方文档](https://www.webpackjs.com/configuration/other-options/#amd)，文档不可解决再看别人的，本文为官方文档重要常用部分摘录，版本 4，部分加入自身理解进行了些修改，手打一遍，仅为巩固记忆

<!-- more -->

# 安装

首先计算机得有 Nodejs 环境，安装方法略

```bash
# 安装webpack和webpack-cli
# 全局,webpack命令可全局使用,官方不推荐
npm install -g webpack webpack-cli
# 当前项文件夹，使用时需要在package.json中配置scripts,然后npm run
npm install --save-dev webpack webpack-cli
```

# 概念

## 入口(entry)

程序入口，把文件依赖看成树状结构，入口就是根节点，从此处开始遍历查找依赖图。把某个文件看作父节点，被依赖的文件即是其子节点

## 出口(output)

output 输出，打包后生成的文件名，位置等

## 解析器(loader)

处理特定类型文件，将其内容转化为 webpack 可理解处理的结构，需要配置的部分为匹配规则和使用的 loader

## 模式(mode)

开发环境 development 或者生产环境 production(production 自动启用 UglifyPlugin 等)

## 插件(plugins)

插件意思不用过多解释，webpack 中插件为一个有 apply 属性的 js 对象，会被 webpack 的 compile 调用，使用时需要 new 创建实例

## 配置

配置文件是 commjs 模块，引用依赖通过 require,编写时可以像编写 js 一样，也可以使用其他语言（详见官方文档），只需要保证配置文件最终输出的是 webpack 要求的特定函数（根据环境动态生成配置），特定对象(一般配置），或特定对象数组(规定多组配置每组配置都生成一个结果)
不建议一个文件里写太长配置，需要分文件

## manifest 和 runtime

manifest 是记录模块信息的部分代码，runtime 是决定如何加载调度模块的部分代码

## 目标(target)

目标平台

## 热替换(hmr)

普通情况，一次代码更改需更新整个依赖图；热替换情况一次代码更新之只更新代码改变了的模块部分

# 配置

<details>
<summary>官网给出的主要配置</summary>

```javascript
const path = require('path');

module.exports = {
  mode: "production", // "production" | "development" | "none"
  mode: "production", // enable many optimizations for production builds
  mode: "development", // enabled useful tools for development
  mode: "none", // no defaults
  // Chosen mode tells webpack to use its built-in optimizations accordingly.

  entry: "./app/entry", // string | object | array
  entry: ["./app/entry1", "./app/entry2"],
  entry: {
    a: "./app/entry-a",
    b: ["./app/entry-b1", "./app/entry-b2"]
  },
  // 这里应用程序开始执行
  // webpack 开始打包

  output: {
    // webpack 如何输出结果的相关选项

    path: path.resolve(__dirname, "dist"), // string
    // 所有输出文件的目标路径
    // 必须是绝对路径（使用 Node.js 的 path 模块）

    filename: "bundle.js", // string
    filename: "[name].js", // 用于多个入口点(entry point)（出口点？）
    filename: "[chunkhash].js", // 用于长效缓存
    // 「入口分块(entry chunk)」的文件名模板（出口分块？）

    publicPath: "/assets/", // string
    publicPath: "",
    publicPath: "https://cdn.example.com/",
    // 输出解析文件的目录，url 相对于 HTML 页面

    library: "MyLibrary", // string,
    // 导出库(exported library)的名称

    libraryTarget: "umd", // 通用模块定义
        libraryTarget: "umd2", // 通用模块定义
        libraryTarget: "commonjs2", // exported with module.exports
        libraryTarget: "commonjs-module", // 使用 module.exports 导出
        libraryTarget: "commonjs", // 作为 exports 的属性导出
        libraryTarget: "amd", // 使用 AMD 定义方法来定义
        libraryTarget: "this", // 在 this 上设置属性
        libraryTarget: "var", // 变量定义于根作用域下
        libraryTarget: "assign", // 盲分配(blind assignment)
        libraryTarget: "window", // 在 window 对象上设置属性
        libraryTarget: "global", // property set to global object
        libraryTarget: "jsonp", // jsonp wrapper
    // 导出库(exported library)的类型

    /* 高级输出配置（点击显示） */

    pathinfo: true, // boolean
    // 在生成代码时，引入相关的模块、导出、请求等有帮助的路径信息。

    chunkFilename: "[id].js",
    chunkFilename: "[chunkhash].js", // 长效缓存(/guides/caching)
    // 「附加分块(additional chunk)」的文件名模板

    jsonpFunction: "myWebpackJsonp", // string
    // 用于加载分块的 JSONP 函数名

    sourceMapFilename: "[file].map", // string
    sourceMapFilename: "sourcemaps/[file].map", // string
    // 「source map 位置」的文件名模板

    devtoolModuleFilenameTemplate: "webpack:///[resource-path]", // string
    // 「devtool 中模块」的文件名模板

    devtoolFallbackModuleFilenameTemplate: "webpack:///[resource-path]?[hash]", // string
    // 「devtool 中模块」的文件名模板（用于冲突）

    umdNamedDefine: true, // boolean
    // 在 UMD 库中使用命名的 AMD 模块

    crossOriginLoading: "use-credentials", // 枚举
    crossOriginLoading: "anonymous",
    crossOriginLoading: false,
    // 指定运行时如何发出跨域请求问题

    /* 专家级输出配置（自行承担风险） */

    devtoolLineToLine: {
      test: /\.jsx$/
    },
    // 为这些模块使用 1:1 映射 SourceMaps（快速）

    hotUpdateMainFilename: "[hash].hot-update.json", // string
    // 「HMR 清单」的文件名模板

    hotUpdateChunkFilename: "[id].[hash].hot-update.js", // string
    // 「HMR 分块」的文件名模板

    sourcePrefix: "\t", // string
    // 包内前置式模块资源具有更好可读性
  },

  module: {
    // 关于模块配置

    rules: [
      // 模块规则（配置 loader、解析器等选项）

      {
        test: /\.jsx?$/,
        include: [
          path.resolve(__dirname, "app")
        ],
        exclude: [
          path.resolve(__dirname, "app/demo-files")
        ],
        // 这里是匹配条件，每个选项都接收一个正则表达式或字符串
        // test 和 include 具有相同的作用，都是必须匹配选项
        // exclude 是必不匹配选项（优先于 test 和 include）
        // 最佳实践：
        // - 只在 test 和 文件名匹配 中使用正则表达式
        // - 在 include 和 exclude 中使用绝对路径数组
        // - 尽量避免 exclude，更倾向于使用 include

        issuer: { test, include, exclude },
        // issuer 条件（导入源）

        enforce: "pre",
        enforce: "post",
        // 标识应用这些规则，即使规则覆盖（高级选项）

        loader: "babel-loader",
        // 应该应用的 loader，它相对上下文解析
        // 为了更清晰，`-loader` 后缀在 webpack 2 中不再是可选的
        // 查看 webpack 1 升级指南。

        options: {
          presets: ["es2015"]
        },
        // loader 的可选项
      },

      {
        test: /\.html$/,
        test: "\.html$"

        use: [
          // 应用多个 loader 和选项
          "htmllint-loader",
          {
            loader: "html-loader",
            options: {
              /* ... */
            }
          }
        ]
      },

      { oneOf: [ /* rules */ ] },
      // 只使用这些嵌套规则之一

      { rules: [ /* rules */ ] },
      // 使用所有这些嵌套规则（合并可用条件）

      { resource: { and: [ /* 条件 */ ] } },
      // 仅当所有条件都匹配时才匹配

      { resource: { or: [ /* 条件 */ ] } },
      { resource: [ /* 条件 */ ] },
      // 任意条件匹配时匹配（默认为数组）

      { resource: { not: /* 条件 */ } }
      // 条件不匹配时匹配
    ],

    /* 高级模块配置（点击展示） */

    noParse: [
      /special-library\.js$/
    ],
    // 不解析这里的模块

    unknownContextRequest: ".",
    unknownContextRecursive: true,
    unknownContextRegExp: /^\.\/.*$/,
    unknownContextCritical: true,
    exprContextRequest: ".",
    exprContextRegExp: /^\.\/.*$/,
    exprContextRecursive: true,
    exprContextCritical: true,
    wrappedContextRegExp: /.*/,
    wrappedContextRecursive: true,
    wrappedContextCritical: false,
    // specifies default behavior for dynamic requests
  },

  resolve: {
    // 解析模块请求的选项
    // （不适用于对 loader 解析）

    modules: [
      "node_modules",
      path.resolve(__dirname, "app")
    ],
    // 用于查找模块的目录

    extensions: [".js", ".json", ".jsx", ".css"],
    // 使用的扩展名

    alias: {
      // 模块别名列表

      "module": "new-module",
      // 起别名："module" -> "new-module" 和 "module/path/file" -> "new-module/path/file"

      "only-module$": "new-module",
      // 起别名 "only-module" -> "new-module"，但不匹配 "only-module/path/file" -> "new-module/path/file"

      "module": path.resolve(__dirname, "app/third/module.js"),
      // 起别名 "module" -> "./app/third/module.js" 和 "module/file" 会导致错误
      // 模块别名相对于当前上下文导入
    },
    /* 可供选择的别名语法（点击展示） */
    alias: [
      {
        name: "module",
        // 旧的请求

        alias: "new-module",
        // 新的请求

        onlyModule: true
        // 如果为 true，只有 "module" 是别名
        // 如果为 false，"module/inner/path" 也是别名
      }
    ],

    /* 高级解析选项（点击展示） */

    symlinks: true,
    // 遵循符号链接(symlinks)到新位置

    descriptionFiles: ["package.json"],
    // 从 package 描述中读取的文件

    mainFields: ["main"],
    // 从描述文件中读取的属性
    // 当请求文件夹时

    aliasFields: ["browser"],
    // 从描述文件中读取的属性
    // 以对此 package 的请求起别名

    enforceExtension: false,
    // 如果为 true，请求必不包括扩展名
    // 如果为 false，请求可以包括扩展名

    moduleExtensions: ["-module"],
    enforceModuleExtension: false,
    // 类似 extensions/enforceExtension，但是用模块名替换文件

    unsafeCache: true,
    unsafeCache: {},
    // 为解析的请求启用缓存
    // 这是不安全，因为文件夹结构可能会改动
    // 但是性能改善是很大的

    cachePredicate: (path, request) => true,
    // predicate function which selects requests for caching

    plugins: [
      // ...
    ]
    // 应用于解析器的附加插件
  },

  performance: {
    hints: "warning", // 枚举
    hints: "error", // 性能提示中抛出错误
    hints: false, // 关闭性能提示
    maxAssetSize: 200000, // 整数类型（以字节为单位）
    maxEntrypointSize: 400000, // 整数类型（以字节为单位）
    assetFilter: function(assetFilename) {
      // 提供资源文件名的断言函数
      return assetFilename.endsWith('.css') || assetFilename.endsWith('.js');
    }
  },

  devtool: "source-map", // enum
  devtool: "inline-source-map", // 嵌入到源文件中
  devtool: "eval-source-map", // 将 SourceMap 嵌入到每个模块中
  devtool: "hidden-source-map", // SourceMap 不在源文件中引用
  devtool: "cheap-source-map", // 没有模块映射(module mappings)的 SourceMap 低级变体(cheap-variant)
  devtool: "cheap-module-source-map", // 有模块映射(module mappings)的 SourceMap 低级变体
  devtool: "eval", // 没有模块映射，而是命名模块。以牺牲细节达到最快。
  // 通过在浏览器调试工具(browser devtools)中添加元信息(meta info)增强调试
  // 牺牲了构建速度的 `source-map' 是最详细的。

  context: __dirname, // string（绝对路径！）
  // webpack 的主目录
  // entry 和 module.rules.loader 选项
  // 相对于此目录解析

  target: "web", // 枚举
  target: "webworker", // WebWorker
  target: "node", // node.js 通过 require
  target: "async-node", // Node.js 通过 fs and vm
  target: "node-webkit", // nw.js
  target: "electron-main", // electron，主进程(main process)
  target: "electron-renderer", // electron，渲染进程(renderer process)
  target: (compiler) => { /* ... */ }, // 自定义
  // 包(bundle)应该运行的环境
  // 更改 块加载行为(chunk loading behavior) 和 可用模块(available module)

  externals: ["react", /^@angular\//],
  externals: "react", // string（精确匹配）
  externals: /^[a-z\-]+($|\/)/, // 正则
  externals: { // 对象
    angular: "this angular", // this["angular"]
    react: { // UMD
      commonjs: "react",
      commonjs2: "react",
      amd: "react",
      root: "React"
    }
  },
  externals: (request) => { /* ... */ return "commonjs " + request }
  // 不要遵循/打包这些模块，而是在运行时从环境中请求他们

  stats: "errors-only",
  stats: { //object
    assets: true,
    colors: true,
    errors: true,
    errorDetails: true,
    hash: true,
    // ...
  },
  // 精确控制要显示的 bundle 信息

  devServer: {
    proxy: { // proxy URLs to backend development server
      '/api': 'http://localhost:3000'
    },
    contentBase: path.join(__dirname, 'public'), // boolean | string | array, static file location
    compress: true, // enable gzip compression
    historyApiFallback: true, // true for index.html upon 404, object for multiple paths
    hot: true, // hot module replacement. Depends on HotModuleReplacementPlugin
    https: false, // true for self-signed, object for cert authority
    noInfo: true, // only errors & warns on hot reload
    // ...
  },

  plugins: [
    // ...
  ],
  // 附加插件列表


  /* 高级配置（点击展示） */

  resolveLoader: { /* 等同于 resolve */ }
  // 独立解析选项的 loader

  parallelism: 1, // number
  // 限制并行处理模块的数量

  profile: true, // boolean
  // 捕获时机信息

  bail: true, //boolean
  // 在第一个错误出错时抛出，而不是无视错误。

  cache: false, // boolean
  // 禁用/启用缓存

  watch: true, // boolean
  // 启用观察

  watchOptions: {
    aggregateTimeout: 1000, // in ms
    // 将多个更改聚合到单个重构建(rebuild)

    poll: true,
    poll: 500, // 间隔单位 ms
    // 启用轮询观察模式
    // 必须用在不通知更改的文件系统中
    // 即 nfs shares（译者注：Network FileSystem，最大的功能就是可以透過網路，讓不同的機器、不同的作業系統、可以彼此分享個別的檔案 ( share file )）
  },

  node: {
    // Polyfills and mocks to run Node.js-
    // environment code in non-Node environments.

    console: false, // boolean | "mock"
    global: true, // boolean | "mock"
    process: true, // boolean
    __filename: "mock", // boolean | "mock"
    __dirname: "mock", // boolean | "mock"
    Buffer: true, // boolean | "mock"
    setImmediate: true // boolean | "mock" | "empty"
  },

  recordsPath: path.resolve(__dirname, "build/records.json"),
  recordsInputPath: path.resolve(__dirname, "build/records.json"),
  recordsOutputPath: path.resolve(__dirname, "build/records.json"),
  // TODO

}
```

</details>

## 多种配置类型

配置文件可以导出函数类型，Promise 对象类型，数组类型

1.  导出函数类型时参数为 env,和其余参数，返回值为配置对象，以此根据参数决定配置
2.  导出 Promise 时同理，异步生成配置
3.  导出数组时每个数组项生成一个结果，每个项可以是函数，Promise,和对象

## context 和入口

### context

项目根路径（绝对路径），默认使用当前路径，官方推荐传值
传值可能会用到 path 对象生成绝对路径，[path 的文档](http://nodejs.cn/api/path.html#path_path_dirname_path)

### entry

`string | [string] | object { <key>: string | [string] } | (function: () => string | [string] | object { <key>: string | [string] })`

从上可见 entry 有如下配置格式

1.  字符串（入口文件）
2.  字符串数组（入口文件列表）
3.  对象（对象的成员值应当是字符串或字符串数组）
4.  返回上述三种类型任意一种的函数，或 promsie

## devtool 选项

`String|false`

devtool 用于配置源代码映射(source map)
devtool 有以下可选值

devtool 不设置或设为 undefined 的话默认表现为 eval 而似乎不像官网说的是不生成 source-map
devtool 设置"",false 这两个 bool 值为 false 的值时，表现为 false 值

| devtool                        | 构建速度 | 重新构建速度 | 生产环境 | 质量                   |
| :----------------------------- | :------- | :----------- | :------- | :--------------------- |
| false 值                       | +3       | +3           | 可选     | 打包后的代码           |
| eval                           | +3       | +3           | 不可选   | 生成后的代码           |
| cheap-eval-source-map          | +1       | +2           | 不可选   | 转译过的代码（仅限行） |
| cheap-module-eval-source-map   | 0        | +2           | 不可选   | 原始源代码（仅限行）   |
| eval-source-map                | -2       | +1           | 不可选   | 原始源代码             |
| cheap-source-map               | +1       | 0            | 不可选   | 转译过的代码（仅限行） |
| cheap-module-source-map        | 0        | -1           | 不可选   | 原始源代码（仅限行）   |
| inline-cheap-source-map        | +1       | 0            | 不可选   | 转译过的代码（仅限行） |
| inlien-cheap-module-source-map | 0        | -1           | 不可选   | 原始源代码（仅限行）   |
| source-map                     | -2       | -2           | 可选     | 原始源代码             |
| inline-source-map              | -2       | -2           | 不可选   | 原始源代码             |
| hidden-source-map              | -2       | -2           | 可选     | 原始源代码             |
| nosources-source-map           | -2       | -2           | 可选     | 无源代码内容           |

### 映射质量的含义

[详见此处](https://www.webpackjs.com/configuration/devtool/#%E5%93%81%E8%B4%A8%E8%AF%B4%E6%98%8E-quality-)
代码映射的质量决定了你在浏览器中调试时能看到 webpack 处理到哪个阶段的代码
代码处理阶段越后，浏览器调试时定位错误越困难，具体区别可以自己配置 devServer，devtool 在浏览器打开 f12 开发者工具转到 sources 查看

分为 4 个阶段

1.  loader 转译前（原始源代码）
    这时的代码映射可以看到源代码的文件结构（模块划分），文件内的代码内容也是源代码内容
2.  转译后，webpack 转换前（转译过的代码）
    这时的代码映射可以看到源代码的文件结构（模块划分），文件内的代码内容是经过 loader 处理过的内容，如果你没配置 loader 的话和阶段 1 是没有区别的。
3.  webpack 转换后，打包前（生成后的代码）
    这时的代码映射仍可以看到源代码的文件结构（模块划分），文件内的代码内容是经过 webpack 转换后的内容，类似 import，之类的语句都被转换为 var module**WEBPACK_IMPORTED_MODULE_1** = **webpack_require**(42);之类的语句
4.  打包后的代码，即为你用 build 打包后产生的那些文件里的代码（打包后的代码）
    这时的代码映射只能看到一个打包结果，文件结构以及内容与你在相同配置下 build 到 dist 文件夹里的相同，无法看到源文件结构（模块划分）

（仅限行）指的是没有列映射。
无源代码内容：只有映射，可以看到源代码文件结构（模块划分），但无法看到源文件中的代码
hidden-source-map 生成了代码映射，但是没有在打包结果中指定映射文件路径，因此在浏览器的开发者工具中无法看到源代码和源文件结构

### 各修饰词意思

source-map:有源代码映射文件（即不存在 eval 和 inline 修饰词时会在代码生成路径生成.map 文件）
cheap:消耗更小的源代码映射（无列映射）
eval:代码和代码映射被 eval 包裹
inline:产生的代码映射数据直接转成 dataurl 打包到生成的 js 中

### 考量因素

1.  对于开发环境，需要不断地重新构建，因此构建速度和重新构建速度是一个重要因素，另外代码映射质量也比较重要。
    因此我认为 cheap-module-eval-source-map 为最佳选择，既有速度又有质量，虽然只有行映射

2.  在生产环境很少需要定位到源代码，不想暴露源代码的结构，并且希望打包出来的 js 文件尽量小
    因此官方建议生产环境 devtool 配置选用：
    false 值，hidden-source-map，nosources-source-map，source-map(选用 source-map 时应当配置服务器不允许普通用户请求你的.map 文件)

### 进阶配置

可以使用 SourceMapDevToolPlugin 进行更细粒度的配置，代替 devtool 的效果。

## output

> output 位于对象最顶级键(key)，包括了一组选项，指示 webpack 如何去输出、以及在哪里输出你的「bundle、asset 和其他你所打包或使用 webpack 载入的任何内容」。
> [中文文档](https://www.webpackjs.com/configuration/output/)

### `output.chunkFilename`

`string = '[id].js'`
此选项决定了非入口(non-entry)(chunk)文件的名称。

请注意，这些文件名需要在运行时生成，以发送对块的请求。 因此，占位符（如`[name]`和`[chunkhash]`）需要在 Webpack 运行时中将从块 ID 到占位符值的映射添加到输出包。 这会增加大小，并且当任何块的占位符值更改时，包可能会失效。(意义不明)

默认是`[id].js`，或者由 filename 转化成的值（filename 中的"[name]"占位符会被替换为"[id]",或者"[name]"前会添加"[id]."

### `output.filename`

`string` `function (pathData, assetInfo) => string`

输出文件名，影响打包产生的文件,对于有多个入口的情况，或者通过插件拆分代码时应当使用模板字符串。

> 注意，此选项不会影响那些「按需加载 chunk」的输出文件。对于这些文件，请使用 output.chunkFilename 选项来控制输出。通过 loader 创建的文件也不受影响。在这种情况下，你必须尝试 loader 特定的可用选项。
> 类似 html-webpack-plugin 等插件产生的文件名也不会被影响。

**模板字符串**

| 模板          | 描述                                                                 |
| ------------- | -------------------------------------------------------------------- |
| [hash]        | 模块标识符(module identifier)的 hash                                 |
| [contenthash] | the hash of the content of a file, which is different for each asset |
| [chunkhash]   | chunk 内容的 hash                                                    |
| [name]        | 模块名                                                               |
| [id]          | 模块标识符                                                           |
| [query]       | 模块文件名后的 query 字符串                                          |
| [function]    | 能返回 string 类型文件名的函数                                       |

当使用[`ExtractTextWebpackPlugin`](/plugins/extract-text-webpack-plugin)插件时, 可以使用 `[contenthash]`提取文件的 hash 值.

### `output.path`

`string = path.join(process.cwd(), 'dist')`

输出路径，格式是绝对路径。

### `output.publicPath`

`string|function`
默认为""
不影响代码输出至哪个文件夹，但影响代码中引用的路径，比如在网页中引用了<em>b.png</em>,<em>b.png</em>被打包至<em>dist/images/</em>，则引用的的地址就变成

```javascript
path.join(publicPath, "images/b.png");
```

同时影响 devServer 的地址根路径。
此选项在需要将程序部署到域名的子路径时配置，程序部署根路径为<em>host/</em>时,一般不用配置.
中文文档说明

> 此选项指定在浏览器中所引用的「此输出目录对应的公开 URL」。相对 URL(relative URL) 会被相对于 HTML 页面（或 <base> 标签）解析。相对于服务的 URL(Server-relative URL)，相对于协议的 URL(protocol-relative URL) 或绝对 URL(absolute URL) 也可是可能用到的，或者有时必须用到，例如：当将资源托管到 CDN 时。

## resolve

路径解析配置

### `resolve.alias`

路径别名，绝对路径
别名后面带\$表示绝对匹配（只匹配和别名完全相同的，不匹配子目录）
样式中使用路径别名需要在路径前加`~`符号

**例子**

| `alias:`                           | `import 'xyz'`                        | `import 'xyz/file.js'`               |
| ---------------------------------- | ------------------------------------- | ------------------------------------ |
| `{}`                               | `/abc/node_modules/xyz/index.js`      | `/abc/node_modules/xyz/file.js`      |
| `{ xyz: '/abc/path/to/file.js' }`  | `/abc/path/to/file.js`                | error                                |
| `{ xyz$: '/abc/path/to/file.js' }` | `/abc/path/to/file.js`                | `/abc/node_modules/xyz/file.js`      |
| `{ xyz: './dir/file.js' }`         | `/abc/dir/file.js`                    | error                                |
| `{ xyz$: './dir/file.js' }`        | `/abc/dir/file.js`                    | `/abc/node_modules/xyz/file.js`      |
| `{ xyz: '/some/dir' }`             | `/some/dir/index.js`                  | `/some/dir/file.js`                  |
| `{ xyz$: '/some/dir' }`            | `/some/dir/index.js`                  | `/abc/node_modules/xyz/file.js`      |
| `{ xyz: './dir' }`                 | `/abc/dir/index.js`                   | `/abc/dir/file.js`                   |
| `{ xyz: 'modu' }`                  | `/abc/node_modules/modu/index.js`     | `/abc/node_modules/modu/file.js`     |
| `{ xyz$: 'modu' }`                 | `/abc/node_modules/modu/index.js`     | `/abc/node_modules/xyz/file.js`      |
| `{ xyz: 'modu/some/file.js' }`     | `/abc/node_modules/modu/some/file.js` | error                                |
| `{ xyz: 'modu/dir' }`              | `/abc/node_modules/modu/dir/index.js` | `/abc/node_modules/modu/dir/file.js` |
| `{ xyz$: 'modu/dir' }`             | `/abc/node_modules/modu/dir/index.js` | `/abc/node_modules/xyz/file.js`      |

### `resolve.extensions`

`[string] = ['.wasm', '.mjs', '.js', '.json']`
自动解析的扩展名，数组
默认是

```javascript
module.exports = {
  resolve: {
    extensions: [".wasm", ".mjs", ".js", ".json"],
  },
};
```

这使得 import 模块时，不需要写这些文件的扩展名，如下

```javascript
import file from "file.js";
// 可以写成
import file from "file";
```

### `resolve.mainFields`

`[string]`
默认值根据 target 的配置有所不同

当 target 属性设置为 webworker, web 或者没有指定，默认值为：

```js
module.exports = {
  //...
  resolve: {
    mainFields: ["browser", "module", "main"],
  },
};
```

对于其他任意的 target（包括 node），默认值为：

```js
module.exports = {
  //...
  resolve: {
    mainFields: ["module", "main"],
  },
};
```

node_modules 里安装的包的 package.json 文件里大多有字段（如'main')指示该到哪找模块
如 webpack 包的 package.json 中就有个,可自行打开 node_modules/webpack/package.json 看看

```json
{
  ...
  "main": "lib/webpack.js",
  ...
}
```

有的依赖包的 package.json 文件中有多个字段指示导入位置
mainFields 决定对这些字段的搜索顺序。
假设一个名为*d3*的包的 package.json 有如下字段

```json
{
  ...
  "main":"index.js",
  "test":"lib/index.js"
  ...
}
```

配置的 mainFields 如下

```js
module.exports = {
  resolve: {
    mainFields: ["test", "main"],
  },
};
```

则当你`import d3 from "d3"`webpack 会优先从该包的 lib/index.js 处引入模块

### `resolve.modules`

`[string] = ["node_modules"]`
在哪寻找模块，默认值是`["node_modules"]`
数组项元素可以是绝对路径和相对路径，是相对路径时，会在当前文件夹，及父级文件夹寻找，如设`["path"]`,会搜索`"./path"`和`"../path"`

### resolve 更多选项

| 名                     | 类型    | 默认值    | 描述                                                                                                         |
| :--------------------- | :------ | :-------- | :----------------------------------------------------------------------------------------------------------- |
| plugins                | array   | 无        | 额外插件                                                                                                     |
| mainFiles              | array   | ["index"] | 文件夹里的主文件，`import "/a/"`时默认变为`import "/a/index"`，类似加载网页路径时默认加载的主页面 index.html |
| enforceModuleExtension | boolean | false     | 是否强制要求使用扩展名                                                                                       |

### `resolveLoader`

与 resolve 对象基本可设置的选项相同，但只用来处理 loader
比如设置

```js
module.exports = {
  resolveLoader:{
    alias:{
      file:"file-loader"
    }
  }
```

你可以在 loader 配置中用 file 代替 file-loader

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.png$/,
        use: [
          //此处本应写file-loader
          "file?name=i/[hash].[ext]",
        ],
      },
    ],
  },
};
```

### `resolveLoader.moduleExtensions`

`array`
处理自动解析后缀的,和 resolve 的 extensions 类似,比如设置了`["-loader"]`,配置 Loader 的时候就可以不写后面的`-loader`

## plugins

`array`
插件列表

## target

`string | function(compiler)`
目标平台，不同的值默认使用了一些不同的插件来转化代码

[支持的字符串设置](https://github.com/webpack/webpack/blob/master/lib/WebpackOptionsApply.js)

| Option              | Description                                                                                                                                                                                                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `async-node`        | Compile for usage in a Node.js-like environment (uses `fs` and `vm` to load chunks asynchronously)                                                                                                                                                                                                   |
| `electron-main`     | Compile for [Electron](https://electronjs.org/) for main process.                                                                                                                                                                                                                                    |
| `electron-renderer` | Compile for [Electron](https://electronjs.org/) for renderer process, providing a target using `JsonpTemplatePlugin`, `FunctionModulePlugin` for browser environments and `NodeTargetPlugin` and `ExternalsPlugin` for CommonJS and Electron built-in modules.                                       |
| `electron-preload`  | Compile for [Electron](https://electronjs.org/) for renderer process, providing a target using `NodeTemplatePlugin` with `asyncChunkLoading` set to `true`, `FunctionModulePlugin` for browser environments and `NodeTargetPlugin` and `ExternalsPlugin` for CommonJS and Electron built-in modules. |
| `node`              | Compile for usage in a Node.js-like environment (uses Node.js `require` to load chunks)                                                                                                                                                                                                              |
| `node-webkit`       | Compile for usage in WebKit and uses JSONP for chunk loading. Allows importing of built-in Node.js modules and [`nw.gui`](http://docs.nwjs.io/en/latest/) (experimental)                                                                                                                             |
| `web`               | Compile for usage in a browser-like environment **(default)**                                                                                                                                                                                                                                        |
| `webworker`         | Compile as WebWorker                                                                                                                                                                                                                                                                                 |

也可是函数，函数的参数 compile 对象，通过调用 compile.apply 并传入使用的插件列表来达到你需要的转化效果
**例**

```js
// 不用插件处理
const options = {
  target: () => undefined,
};
// 自定义使用的插件
const webpack = require("webpack");
const options = {
  target: (compiler) => {
    compiler.apply(
      new webpack.JsonpTemplatePlugin(options.output),
      new webpack.LoaderTargetPlugin("web")
    );
  },
};
```

## module

### `module.noParse`

`RegExp | [RegExp] | function`
loader 不处理匹配到的文件

### `module.rules`

`[(Object Rule)]`
Rule(处理规则)对象列表

### Rule

处理规则对象
分为三部分：条件，结果，和嵌套规则
**rule 条件**
resource：被请求文件的绝对位置。issuer：文件请求者的绝对位置。
**Condition**
条件

> 条件(Condition)可以是这些之一：
>
> 1.  字符串：用来匹配以提供的字符串为开头的文件或文件夹路径。
> 2.  正则表达式
> 3.  函数：根据参数计算匹配结果，返回真值时表示匹配成功。
> 4.  条件数组：至少匹配一个条件。
> 5.  对象：匹配所有属性。每个属性都有一个定义行为如下。
>     { test: Condition }匹配特定条件。一般是提供一个正则表达式或正则表达式的数组，但这不是强制的。
>     { include: Condition }：匹配特定条件。一般是提供一个字符串或者字符串数组，但这不是强制的。
>     { exclude: Condition }：排除特定条件。一般是提供一个字符串或字符串数组，但这不是强制的。
>     { and: [Condition] }：必须匹配数组中的所有条件
>     { or: [Condition] }：匹配数组中任何一个条件
>     { not: [Condition] }：必须排除这个条件

**rule 结果**
匹配到文件后如何处理。

**Rule 属性如下**

| 名称          | 类型              | 描述                                                                                                                                                                                                                                                                             |
| :------------ | :---------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| enforece      | `"pre"` `"post"`  | 影响 loader 类型,按使用顺序排序分为前置，行内，普通，和后置，不设置时为普通所有普通 loader 可以通过在请求中加上 ! 前缀来忽略（覆盖）。所有普通和前置 loader 可以通过在请求中加上 -! 前缀来忽略（覆盖）。所有普通，后置和前置 loader 可以通过在请求中加上 !! 前缀来忽略（覆盖）。 |
| exclude       | `Condition`       | loader 不处理匹配到的 resource                                                                                                                                                                                                                                                   |
| include       | `Condition`       | loader 处理匹配到的 resource                                                                                                                                                                                                                                                     |
| issuer        | `Condition`       | loader 处理匹配到的 issuer 所请求的 resource                                                                                                                                                                                                                                     |
| loader        | `String`          | `use:[{loader}]`的简写                                                                                                                                                                                                                                                           |
| oneOf         | `[(object Rule)]` | 嵌套规则，只匹配一个规则，优先级从前往后                                                                                                                                                                                                                                         |
| parse         | `Object`          | 启用或禁用解析器,[详见此](https://www.webpackjs.com/configuration/module/#rule-parser)                                                                                                                                                                                           |
| resource      | `Object`          | 包含各种条件，include,exclude 及其他都是 resource.include,resource.exclude 的简写，详见下文[resource 对象的内容](#resource对象的内容)                                                                                                                                            |
| resourceQuery | `Condition`       | 请求路径中的 query 字符串                                                                                                                                                                                                                                                        |
| rules         | `[(Object Rule)]` | 嵌套规则                                                                                                                                                                                                                                                                         |
| test          | `Condition`       | loader 处理匹配到的 resource                                                                                                                                                                                                                                                     |
| use           | `[(UseEntry)]`    | UseEntry 列表,每个 UseEntry 指定一个 loader,`use:["style-loader"]`是`use:[{loader:"style-loader"}]`的简写形式                                                                                                                                                                    |

**UseEntry 定义**
必须有一个 String 类型的 loader 属性，指明使用哪个 loader
options 属性可以设置用来为 Loader 添加额外的设置

### resource 对象的内容

{ test: Condition }：匹配特定条件。一般是提供一个正则表达式或正则表达式的数组，但这不是强制的。
{ include: Condition }：匹配特定条件。一般是提供一个字符串或者字符串数组，但这不是强制的。
{ exclude: Condition }：排除特定条件。一般是提供一个字符串或字符串数组，但这不是强制的。
{ and: [Condition] }：必须匹配数组中的所有条件
{ or: [Condition] }：匹配数组中任何一个条件
{ not: [Condition] }：必须排除这个条件

## devServer

只记录常用配置，[其余配置请看此](https://www.webpackjs.com/configuration/dev-server/#devserver)

### `devServer.after`

`function`
提供在所有其他中间件执行完成后执行自定义中间件的功能

### `devServer.before`

`function`
提供在其他所有中间件执行之前执行自定义中间件的功能，者可以被用来定义自定义请求处理器
例如

```js
before(app){
  app.get('/some/path', function(req, res) {
    res.json({ custom: 'response' });
  });
}
```

### `devServer.clientLogLevel`

`string`
输出提示信息级别

### `devServer.compress`

`boolean`
是否启用 gzip 压缩

### `devServer.contentBase`

`boolean` `string` `array`
不需要编译的静态文件的位置

### `devServer.filename`

`string`
lazy mode 下每个请求会产生全新的编译结果，使用 filename,可以指只某个文件被请求时编译。

### `devServer.hot`

`boolean`
是否启用热模块替换。

模块热替换并不像网上的博客说得那么简单，改个配置 jia 插件就行，不设置好的话实际上就是没有的功能（仍会全局刷新，或修改文件后直接不更新）。

js 的热替问题换参看[模块热替换](https://www.webpackjs.com/guides/hot-module-replacement/#%E9%97%AE%E9%A2%98)。

对于特定框架来说还需要做一些额外设置来配合 [其他代码和框架](https://www.webpackjs.com/guides/hot-module-replacement/#%E5%85%B6%E4%BB%96%E4%BB%A3%E7%A0%81%E5%92%8C%E6%A1%86%E6%9E%B6)。

**例 react webpack 项目启用热替换**:
[官方文档](https://github.com/gaearon/react-hot-loader)
事实上以下配置有时会失灵（某些情况下修改文件后界面仍会全局刷新而不是替换局部组件），由于学习不够深入，可能配置中有一些不对或不足，仅供参考。

1. 设置`devServer.hot`为`true`
2. 使用`react-hot-loader`处理 jsx 文件，配置如下
   ```js
   module.exports = {
     module: {
       rules: [
         {
           exclude: /node_modules/,
           use: ["react-hot-loader/webpack", "babel-loader", "eslint-loader"],
           test: /\.jsx|\.js$/,
         },
       ],
     },
     plugins: [
       // 非必要
       new webpack.NamedModulesPlugin(),
       new webpack.HotModuleReplacementPlugin(),
     ],
   };
   ```
3. 在入口文件代码前插入
   ```js
   if (module.hot) {
     module.hot.accept();
   }
   ```
4. babel 配置文件中加入`react-hot-loader/babel`插件
5. 再修改入口(是否需要存疑)
   ```js
   module.exports = {
     entry: {
       App: ["react-hot-loader/patch", "./src/main.jsx"],
     },
   };
   ```
6. 再在程序入口文件修改如下
   ```jsx
   import App from "@/router/";
   ReactDom.render(
     <AppContainer>
       <App />
     </AppContainer>,
     document.getElementById("app")
   );
   ```

[官方文档此处](https://www.webpackjs.com/configuration/dev-server/#devserver-hot)有如下说明，我不确定我是否理解错了意思，毕竟[官网给出的配置](https://www.webpackjs.com/guides/hot-module-replacement/#%E5%90%AF%E7%94%A8-hmr)设置了`hot`为`true`仍需要在 webpack 配置中添加`HotModuleReplacementPlugin`，而下文却说不用。

> Note that webpack.HotModuleReplacementPlugin is required to fully enable HMR. If webpack or webpack-dev-server are launched with the --hot option, this plugin will be added automatically, so you may not need to add this to your webpack.config.js. See the HMR concepts page for more information.

> 需要使用 webpack.HotModuleReplacementPlugin 来完全启用 HMR。如果 webpack 或者 webpack-dev-server 通过--hot 选项启动，这个插件会被自动添加，此时你不需要把这个添加到你的 webpack.config.js。查看 HMR 概念页面来获取更多信息。

### `devServer.inline`

`boolean`
内联模式还是 Iframe 模式，默认时内联模式。inline 模式构建消息会出现在浏览器控制台。iframe 的构架消息出现在 iframe 中.模块热替换启用时，建议使用内联模式

### `devServer.lazy`

`boolean`
启用时，只有在请求时包才会被编译，watchOptions 在启用 lazy mode 时无效

### `devServer.open`

`boolean` `string`
启动时是否打开浏览器，可以指定浏览器名称

### `devServer.openPage`

`string` `array`
打开浏览器时默认打开的路径,可以是多个

### `devServer.port`

`Object`
服务端口

### `proxy`

`Object` `array`
指定某些请求的代理服务地址，可以是对象，也可以是对象列表，对象配置格式为

```js
// 简单格式
module.exports = {
  proxy: {
    "/api": "http://localhost:3000"
  }
}
// 对代理的路径进行详细配置
// 重写请求路径
module.exports = {
  proxy: {
  "/api": {
    target: "http://localhost:3000",
    pathRewrite: {"^/api" : ""}
  }
}
// 不进行https的证书验证
module.exports = {
  proxy: {
  "/api": {
    target: "https://other-server.example.com",
    secure: false,
    // 换源，此时请求和响应不同源
    changeOrigin:true
  }
}
// 对请求进行一些过滤处理
// bypass返回false时，直接返回404错误
// bypass返回null和undefine时，继续请求代理服务
// bypass返回路径字符串，则直接响应对应路径的文件而不请求代理服务
// bypass有个很不错的作用，将所有的路径请求转为对特定页的请求，可以确保测试环境刷新时请求路径不会出错：如vue项目中的路由使用history模式时，非根路径刷新时会导致没加载到页面文件，用bypass就可以解决
module.exports = {
  //...
  devServer: {
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        bypass: function(req, res, proxyOptions) {
          if (req.headers.accept.indexOf('html') !== -1) {
            console.log('Skipping proxy for browser request.');
            return '/index.html';
          }
        }
      }
    }
  }
};
// 当需要代理根路径时，需要指定index为假值

module.exports = {
  //...
  devServer: {
    index: '', // specify to enable root proxying
    host: '...',
    contentBase: '...',
    proxy: {
      context: () => true,
      target: 'http://localhost:1234'
    }
  }
};
```

### `devServer.publicPath`

`string`
决定文件所在位置,例如打包成 bundle.js,则请求时需请求 publicPath/bundle.js
可以是完整的 url 路径

### `devServer.watchOptions`

`Object`
一些监视文件相关的配置，[更多设置](https://www.webpackjs.com/configuration/watch)

```js
// 使用轮询来监听文件变化
module.exports = {
  watchOptions: {
    // 也可以是整数指定轮询间隔毫秒数
    poll: true,
  },
};
```

## externals

`string` `array` `object` `function`
指定外部扩展
指定一些引用的包不会被打包到 bundle 中，而是在运行时获取
例如，从 CDN 引入 jQuery，而不是把它打包：
配置

```js
module.exports = {
  externals: {
    jquery: "jQuery",
  },
};
```

然后在 html 中

```html
<script
  src="https://code.jquery.com/jquery-3.1.0.js"
  integrity="sha256-slogkvB1K3VOkzAI8QITxV3VzpOnkeNVsKvtkYLMjfk="
  crossorigin="anonymous"
></script>
```

这样在 js 中引用 jquery 模块就不需要在本地安装 jquery 库，而是会在运行时在全局变量中寻找变量 jQuery

```js
import $ from 'jquery';

$('.my-element').animate(...);
```

### 外部库可以通过以下形式引入

root: 外部库是全局变量
commonjs: 外部库是 commonjs 模块
commonjs2: 外部库是 commonjs2 模块
amd: 外部库是 amd 模块

### 配置为 string 类型

例子如上文指出
另一个例子

配置

```js
module.exports = {
  externals: {
    jquery: "commonjs2 jQuery",
  },
};
```

效果如下

```js
import fs from "fs-extra";
// 由于配置中指定了commonjs环境,上述代码会被打包为如下
const fs = require("fs-extra");
```

### [string]类型

引入模块的部分成员

```javascript
module.exports = {
  //...
  externals: {
    subtract: ["./math", "subtract"],
  },
};
import subtract from "subtract";
// 被打包成
subtract = require("./math").subtract;
```

### Object 类型

指定不同系统下的访问方式

```js
module.exports = {
  //...
  externals: {
    react: "react",
  },

  // or

  externals: {
    lodash: {
      commonjs: "lodash",
      amd: "lodash",
      root: "_",
    },
  },

  // or

  externals: {
    subtract: {
      root: ["math", "subtract"],
    },
  },
};
```

### function 类型

自定义筛选一些外部模块如何加载

### regex 类型

排除所有匹配到的模块

## performance

性能提示配置

## node

是否在浏览器中添加 Nodejs 的 api 垫片

## stats

配置命令行显示与不显示哪些信息

# 插件

[插件文档](https://www.webpackjs.com/plugins/)

## splitChunkPlugin

**说明：**

- 一个块指的是某个文件及其依赖（非异步）组成的整体，也指一个块被分割后的每个结果。
- 初始块指的是入口和入口的依赖组成的整体。
- 按需（异步）块指得是通过`import()`,`require.ensure`等方式引入的块，这些块在页面中表现为在第一次使用到时才请求内容，所以称为按需块。
- 入口：配置文件中的 entry

**举例**

如下依赖结构 index.js，index3.js 两个文件为一个初始块
如下依赖结构 index2.js，index3.js，index4.js 三个文件为一个按需块

```js
// index.js
import("./index2.js");
import "./index3.js";
const str = "index.js";

// index2.js
import "./index3.js";
import "./index4.js";
const str = "index2.js";

// index3.js
const str = "index3.js";

// index4.js
const str = "index4.js";
```

**配置说明**

为便于理解，配置可以分为四种

1. 匹配规则
   - 这种配置用于决定哪些文件或块包括在缓存组中、适用生成规则
   - test，chunks，minChunks，type 属于此列
2. 生成规则
   - 这种配置决定新块如何产生，生成的文件名
   - name，filename，enforce，automaticNameDelimiter 属于此列
3. 限制条件
   - 在根据生成规则得出分割结果后再根据限制条件进行一定调整，不影响缓存组匹配文件
   - minSize，maxAsyncRequests，maxInitialRequests 属于此类

**配置列表**

| 配置                                                      | 类型                          | 取值                                                                     | 说明                                                                                                                                                                                                                                                    |
| :-------------------------------------------------------- | :---------------------------- | :----------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `optimization.splitChunks`                                | `Object`                      |                                                                          | 配置对象                                                                                                                                                                                                                                                |
| `splitChunks.automaticNameDelimiter`                      | `String`                      | `any`                                                                    | 分隔符                                                                                                                                                                                                                                                  |
| `splitChunks.chunks`                                      | `String` `Function`           | `'async'` `'initial'` `'all'` `function (chunk)`                         | 用来匹配块，字符串取值分别对应按需块，初始块，全部；函数返回值为 true 时为匹配正确，每有一个块执行一次                                                                                                                                                  |
| `splitChunks.maxAsyncRequests`                            | `Number`                      | `(0,Infinity)`                                                           | 最大并行异步请求数：当使用到某个按需块，会进行一组请求，来加载这个按需块。`maxAsyncRequests`这个选项指示完成这样一次加载最多发送多少次请求（表现为将这个按需块最多分为多少个包）                                                                        |
| `splitChunks.maxInitialRequests`                          | `Number`                      | `(0,Infinity)`                                                           | 一个入口的最大并行请求数，与前述异步块最大并行请求数同理                                                                                                                                                                                                |
| `splitChunks.minChunks`                                   | `Number`                      | `(0,Infinity)`                                                           | 最小被依赖数，对于通过`import ...`,`require()`等方式引入的依赖，最少需要被几个入口或者几个按需块引入才可被缓存组匹配到                                                                                                                                  |
| `splitChunks.name`                                        | `Boolean` `String` `Function` | `true` `function (module, chunks, cacheGroupKey) => string` `any string` | 分割出来的块的名字，若设为静态字符串会导致分割出的块都被打包到一个文件中，设为函数时缓存组匹配到的每个块中包含的每个文件都会使 name 函数执行一次，可通过 name 中返回不同的值，来将块中的每个文件单独分割出来                                            |
| `splitChunks.cacheGroups`                                 | `Object`                      |                                                                          | 缓存组对象，可以使用`splitChunks`对象的所有选项，并且`test`, `priority`和`reuseExistingChunk`只能在缓存组中设置，若某些选项不设置，默认会采用`splitChunks`对象中的设置                                                                                  |
| `splitChunks.cacheGroups.{cacheGroup}.priority`           | `Number`                      | `(-Infinity,Infinity)`                                                   | 优先级，当两个缓存组同时匹配到一个块，会先分出优先级高的那个缓存组的块，剩余再根据优先级低的缓存组规则分块，不填为 0                                                                                                                                    |
| `splitChunks.cacheGroups.{cacheGroup}.reuseExistingChunk` | `Boolean`                     | `true` `false`                                                           | 如果当前块包含已经从主包中分割出来的模块，这个模块将会被复用，而不是重新生成。这会影响生成的块的名称。                                                                                                                                                  |
| `splitChunks.cacheGroups.{cacheGroup}.type`               | `String`                      | `any string`                                                             | 允许根据模块类型分配模块到一个缓存组。                                                                                                                                                                                                                  |
| `splitChunks.cacheGroups.{cacheGroup}.test`               | `RegExp` `String` `Function`  | `any reg` `any string` `function (module, chunk) => boolean`             | 精确匹配路径，块名等，每有一个文件执行一次                                                                                                                                                                                                              |
| `splitChunks.cacheGroups.{cacheGroup}.filename`           | `String` `Function`           | `any string` `function (pathData, assetInfo) => string`                  | 仅对初始块使用，覆盖输出文件名                                                                                                                                                                                                                          |
| `splitChunks.cacheGroups.{cacheGroup}.enforce`            | `Boolean`                     | `true` `false`                                                           | 忽略限制，总是为当前缓存组生成一个块                                                                                                                                                                                                                    |
| `splitChunks.minSize`                                     | `Number`                      | `(0,Infinithy)`                                                          | 限制分割出的块的最小体积,不符合条件则不分割                                                                                                                                                                                                             |
| `splitChunks.maxSize`                                     | `Number`                      | `(0,Infinithy)`                                                          | 指示当某个块大于`maxSize`且分割不会造成产生违反`minSize`的选项时，将块分为更小的块，`splitChunks.maxAsyncSize`，`splitChunks.maxInitialSize`效果与`maxSize`选项相同只是作用范围不同。适用优先级`maxInitialRequest/maxAsyncRequests < maxSize < minSize` |

### 分离 runtime

可直接配置（webpack4 以上可行其余不知可否）

```js
module.exports = {
  // ...
  optimization: {
    runtimeChunk: {
      name: "runtime",
    },
  },
  // ...
};
```

### 示例 1，maxAsyncRequests 使用

```js
// index.js
import("./index2");
const str = "this is index.js";

// index2.js
import "./index3";
import "./index4";
import "./index5";
const str = "this is index2.js";

// index3.js
const str = "this is index3.js";

// index4.js
const str = "this is index4.js";

// index5.js
const str = "this is index5.js";

// webpack.config.js
module.exports = {
  // ...
  optimization: {
    // 将runtime代码分离出去方便查看打包后的代码
    runtimeChunk: {
      name: "runtime",
    },
    splitChunks: {
      // 影响所有缓存组，也可以在缓存组里设置
      maxAsyncRequests: 2,
      cacheGroups: {
        // 关闭默认缓存组
        default: false,
        asyncChunk: {
          chunks: "async",
          // 用不同name将一个块里的文件拆分出来
          name(module, chunks, cacheGroupKey) {
            return "asyncChunk" + Math.floor(Math.random() * 1000);
          },
          minSize: 0,
        },
      },
    },
  },
  // ...
};
```

- 如上 index.js 中按需引入了 index2.js，index2.js 依赖 index\[3|4|5\].js。
- 所以 index.js 在运行到引入 index2.js 的代码处时，会进行请求，以获取 index\[2345\].js 这四个文件包含的代码。这四个文件视作为一个**按需块 A**。
- 由于设置了 minSize = 0，chunks = 'async'，以及 name 函数，只要生成的文件大于 0byte 的都会被从**按需块 A**中分离出来生成单独文件，index\[2|3|4|5\]都满足这个条件。
- 但由于 maxAsyncRequests = 2，限制了 index.js 按需引入**按需块 A**时最大只能进行两次并行请求，因此只有 index2.js 会被从**按需块 A**分离出来。故而**按需块 A**分割成两个文件，一个文件包含了**按需块 A**剩余代码即 index\[345\].js 的代码，一个文件包括 index2.js 代码。index.js 引入**按需块 A**时会并行请求这两个文件。
- 当 maxAsyncRequests 改为 3，index2.js，index3.js 会被从块中分离成单独的文件。其他情况依次类推，文件被分离的优先级似乎是依照文件名的字符串比较结果决定的，即若存在一个 a.js 最先被分离出来的将不是 index2.js 而是 a.js。
- 将上述配置所有 async 改为 initial，maxAsyncRequests 改为 maxInitialRequests，以及代码`import("./index2.js)`改为`import "./index2.js"`后，产生的分块效果相同

### 示例 2，minChunks 使用

```js
// index.js
import("./index2");
import("./index3");
const str = "this is index.js";

// index2.js
import "./index4";
import "./index5";
const str = "this is index2.js";

// index3.js
import "./index4";
const str = "this is index4.js";

// index4.js
const str = "this is index4.js";

// index5.js
const str = "this is index5.js";

// webpack.config.js
module.exports = {
  // ...
  optimization: {
    // 将runtime代码分离出去方便看打包后的代码
    runtimeChunk: {
      name: "runtime",
    },
    splitChunks: {
      // 影响所有缓存组，也可以设置到单个缓存组里
      maxAsyncRequests: 2,
      cacheGroups: {
        // 关闭默认缓存组
        default: false,
        asyncChunk: {
          chunks: "async",
          // 匹配任何块
          minSize: 0,
          minChunks: 2,
        },
      },
    },
  },
  // ...
};
```

- 如上，index.js 有两个按需引入分别为包含 index\[245\].js 的(按需块 A)，和包含 index\[34\].js 的(按需块 B)
- minChunks 设为 2，只有 index4.js 同时被两个按需块依赖，故而 index\[2345\]总共生成三个文件，(按需块 A)包含 index\[25\].js，(按需块 B)包含 index3.js，(共享块 C)包含 index4.js
- 上述结果同时满足 maxAsyncRequests=2，即 index.js 运行到`import("index2.js")`时并行请求(按需块 A)和(共享块 C)，不大于二
- 如果 maxAsyncRequests=1，会被优先满足，index4.js 也不会被分离出来，而是同时包含在(按需块 A)和(按需块 B)中。
- 将上述配置 async 改为 initial，并去除入口 index.js，然后将 index2.js，index3.js 设为入口得到效果相同

### test 函数类型，chunks 函数类型，name 函数类型，等使用

- 开启 debug 模式

  ```bash
  # 全局安装node-nightly
  npm i -g node-nightly
  # 运行
  node-nightly --inspect-brk ./node_modules/webpack/bin/webpack.js
  ```

**test,chunks,name 为函数时执行规则**

以下执行规律为 debug 模式（用 node-nightly 和 chrome://inspect）测出，可能存在理解错误

1. 对每个文件，test 最先执行一次，chunk 在 test 之后执行，name 最后执行
2. 每有一个 js 文件，test 会执行一 ���，传入参数`module`为请求的文件信息，`chunks`为该文件所属的块。
3. name 执行规律如下：
   - 当 test 匹配到文件 index.js，且 index.js 属于块 A，则 name 会执行 1 次，传入的参数是描述 index.js 的 module 对象、chunks 列表`[chunkA]`、缓存组键名。
   - 当 test 匹配到文件 index2.js，且 index2.js 属于块 A 和块 B，则 name 会依次执行 3 次，每次传入的参数第一个是 module 对象；第二个为 chunks 列表，三次执行时值依次是`[chunkA, chunkB]`，`[chunkA]`，`[chunkB]`(前提是 chunks 函数总是返回 true)；第三个为缓存组键名。
4. chunks 执行方式比较怪异，不同时设置 test，name 为函数根本看不出这玩意是按什么规律执行的;也不知道它的 chunk 参数是干嘛的，传了也是白传，基本上能获取到的信息只有 chunk.name，其他不是 undefined 就是 null，按需块连 name 都是 null。chunks 在产生 name 之前执行，执行规则如下：
   - 在每次 name 执行前根据条件执行，取决于 name 函数的参数 chunks 列表，遍历 chunks 列表，列表每个 chunk 元素执行一次 chunks 函数。
   - 具体规律举例如下：
   - 时间 1：对 index.js 属于块 A，即`chunks列表 = [chunkA]`，name 执行之前 chunks 会执行一次，传入块 A 的描述对象；
   - 时间 2：对于 index2.js 属于块 A 块 B，name 执行 3 次。name 第一次执行时`chunks列表 = [chunkA, chunkB]`，chunks 函数遍历 chunks 列表执行两次；name 第二次执行时`chunks列表 = [chunkA]`，则不会执行 chunks 函数，而是直接使用时间 1 的结果；name 第三次执行时`chunks列表 = [chunkB]`，chunks 又会执行一次，因为之前不存在`chunks列表 = [chunkB]`的情况。

虽然上述规律大部分时候用不到，但了解一下总没坏处

## HtmlWebpackPlugin

[文档](https://github.com/jantimon/html-webpack-plugin#third-party-addons)

## HotModuleReplacementPlugin

启用只需要

```js
//使用默认配置
new webpack.HotModuleReplacementPlugin();
// devServer中只需要配置hot选项为true,似乎能自动启用本插件不用配置
```

## ExtractTextPlugin

此配置和模块热替换冲突
webpack4.0 以上改用 mini-css-extract-plugin

## SpritesmithPlugin

用于生成雪碧图，[文档](https://github.com/mixtur/webpack-spritesmith)

# loader

## postcss-loader

用来预处理样式，加浏览器前缀之类的
[文档](https://www.npmjs.com/package/postcss-loader)

## style-loader

将样式添加到 html 文件头部的`<style>`标签中
[文档](https://www.webpackjs.com/loaders/style-loader/)
