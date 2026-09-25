---
title: WebGPU初探索-创建第一个程序
tags:
- WebGPU
- javascript
date: 2023/05/16
---

趁有空尝试一下WebGPU，提前体验了解一下

# 启用webGPU

113版本chrome，在地址栏输入`chrome://flags`

搜索WebGpu并启用

![image-20230516003046701](.\images\image-20230516003046701.png)

# 工程框架

当然可以直接写js文件，不过为避免体验过程中代码越来越复杂变得混乱，还是使用一个简易框架 vite+typescript

## typescript启用webGPU声明

参照gitHub上readme指示，

安装`@webgpu/types`后，配置tsconfig的`typeRoots`

```bash
npm install --save @webgpu/types
```

```json
{
  // ...
  "compilerOptions": {
    // ...
    "typeRoots": [ "./node_modules/@webgpu/types", "./node_modules/@types"]
  }
}
```

## vscode支持wgsl高亮

安装插件

![image-20230516005533896](.\images\image-20230516005533896.png)