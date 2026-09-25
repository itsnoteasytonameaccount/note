---
title: hexo的使用技巧
date: 2018/04/02 11:06:19
tags:
- hexo
---

# hexo 博客使用（一）

## 前言

首先是官方中文文档，不管什么情况，应该养成首先看官方文档的习惯，官方文档解决不了再合理利用搜索引擎
[官方文档点击这里](https://hexo.io/zh-cn/docs/)

<!--more-->

## 基础知识

1.  文章位置
    写文章用的是 markdown，文章路径默认为 source 文件夹中的\_posts,文章文件格式为.md 后缀,至于 markdown 语法[点这里查看](http://wowubuntu.com/markdown/index.html)，当你在控制台执行*hexo generate*后，默认会在 public 目录下生成 html 文件，最终你的博客页面显示的就是这些 html 文件。
2.  主题
    hexo 有着大量的主题，你可以在[官方的主题页面](https://hexo.io/themes/)寻找适合自己的主题,主题的位置默认在 themes 文件夹里，
    主题的安装只需要将包下载在主题文件夹即可，切换主题只需要在\_config.yml 配置文件中，更改主题配置。切换主题后，需要重新 generate
3.  配置文件
    hexo 配置文件为\_config.yml，重要的几个参数有：

    > title:主页的标题
    > author:文章作者
    > root:网页的根路径,比如：若你将页面部署在*www.yoursite.com/root/*下，且*www.yousite.com*是你网站的根路径，你的 root 就应该设为/root/
    > url:网站路径
    > public_dir:文件生成的目录
    > source_dir:文章目录
    > theme:选择主题

    当然，还有许多参数，另外，当你安装了某些插件时，也需要在\_config.yml 中为其配置参数
    有些主题也需要为其配置参数,另外，一些主题还有自己独立的配置文件

4.  第三方服务
    由于 hexo 博客完全没有后台服务，因而有一些需要数据记录和操作的服务是无法做到的，例如记录文章信息供用户检索，或者评论功能，这时就需要第三方服务，网上有许多免费的第三方数据服务，我用过或了解过的有，[algolia](https://www.algolia.com/)(提供数据存储以及检索接口),[友言](http://www.uyan.cc/)(提供社会化分享，评论服务,但似乎所有接口都不支持 https 协议)，[畅言](https://changyan.kuaizhan.com/)(也是评论服务，看过许多网站使用这一服务),[livere](https://livere.com/)(同样是评论服务，支持 https 协议，因而我当前采用的是这一评论服务),另外还有百度 api 等——我的分享功能用的是百度 api

    有许多主题自身接入了第三方服务，要使用只需要在相应配置文件中添加相应配置即可

## 进阶

### 修改主题

他人的主题或许满足不了自身的所有需求，但是自己写主题的时间成本又不小，因而可以采取一种折衷的办法，修改现成的主题，将不满意的地方改为自己喜欢的样子，或为主题添加第三方服务

主题由各部分的模板文件组成，模板文件有两种类型，一种是类似于 jsp 的，通过`<%标签开头%><%标签结尾%>`来嵌入预执行代码的模板语句，后缀为 ejs。

另一种类似`AngularJs`的将代码放在两对花括号之间，本人只进行过第一种主题模板的修改。两种模板需要不同的插件生成 html 文件的。除了少许的差异外，两种模板的语法是基本一致的。

详见官方文档，不作赘述。

### 插件

npm 上有许多关于 hexo 的插件，用以丰富其功能，如果这些插件满足不了你，可以尝试自己写需要的功能插件。

hexo 为所有插件提供 hexo 全局变量
hexo 插件要采用注册器拓展自身功能，举例如下

```javascript
hexo.extend.generator.register("name", functionName);
//generator注册的函数必须返回一个对象，否则generate时可能会报错，具体返回格式参看官方文档
hexo.extend.console.register("name", "describe", name);
```

hexo 可拓展的部分有

> 控制台(console)，此部分的扩展会添加在控制台 hexo 指令的可选参数中

> 生成器(generator),此部分的扩展会在你执行控制台的 hexo generate 指令时自动执行

> 过滤器(filter),某些特定时段对文件内容修改

更多可参看[官方中文 api 文档](https://hexo.bootcss.com/api/filter.html)，另外我尝试写过一个 [hexo-algolia-helper](https://github.com/flyingfishman/hexo-algolia-helper) 插件，虽然代码难看，功能也不好，但应该可作为示例

要写插件，如果是是一些小功能的话可以直接写在 scripts 文件夹中，如果 hexo 博客目录没有 scripts 文件夹，自己添加，scripts 文件夹里的代码文件，会在每次调用控制台时自动执行。

**script 示例：使用相对路径的图片**

文件结构如下

![图片无法显示](../image/struct.png)

则可以在 markdown 文件中使用相对路径

```markdown
![图片无法显示](../image/你的图片路径)
```

`hexo generate`时 image 文件夹会被复制到 public 文件夹下

因此需要写路径函数用来改写路径，在 filter 中注册

```javascript
function postImages(post) {
  post.content = post.content.replace(/<img[^>]*src="[^"]*"[^>]*>/g, function (dir, index) {
    return dir.replace(/\.\.\//, hexo.config.root);
  });
  return post;
}
hexo.extend.filter.register("after_post_render", postImages);
```

如果是大型插件，可以写一个独立插件发布至 npm 上，在通过 npm 安装至 hexo 博客目录下，插件名必须是 hexo-**\***的格式，否则 hexo 不会自动执行该插件

> 要想写好插件，也是最好多看文档，不要盲目信百度，说不定有很多人就是像我一样，学了点三脚猫功夫就出来摆弄。
