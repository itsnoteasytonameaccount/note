---
title: 代码技巧
tags: javascript
date: 2018/04/06 16:25:35
---

# 代码技巧

觉得有意思的一些以前没用过的方法
<!-- more -->

## 短路与和短路或

查了一下短路与好像是是专用于java的名词，不过大多数语言都有这个特性
<!--more-->
1.  模仿断言
	```javascript

	//以下代码只有在debug置为真时&&符号后的代码才会被调用
	debug && a === b && console.log("a equal b");

	//python里大概是这样
	debug=True
	debug and statement and print("test")
	```
2.  条件语句
    也可以用短路与来模仿条件语句
    某些情况简洁和灵活
    同样适用于许多语言
    ```javascript
    //例，用括号转化赋值语句为表达式
    i !== 0 && (b = i/2);
    //等效
    i === 0 || (b = i/2);

    doSomethingAndReturn() && whenReturnFalse();
    //等效
    !doSomethingAndReturn() || whenReturnFalse();
    ```

## array用来补零

```javascript
//补零对齐
function align(str,n){
    let result;
    //新建array对象，在空数组元素之间插0，与要对齐的字符串组装
    n < 0 && (result = (new Array(Math.abs(n) + 1).join("0") + str).slice(n))
    	|| n > 0 && (result = (str + new Array(n + 1).join("0")).slice(0,n));
    //截取n个字符后返回
    return result;
}
```

es6可以直接用`padStart`