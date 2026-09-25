---
title: c#点net笔记
tags:
- CSharp
date: 2018/11/26
---

最近学c#的笔记
<!-- more -->

# iis中部署

在启用或关闭Windows功能中

需要安装了.net支持,版本视情况自定。

另外需要在以下路径勾选.net相关服务
internet information server-->万维网服务-->应用程序与开发功能

在iis中部署后若出现版本错误
在应用程序池中查看是否有需要的版本.net
之后在网站上右击-->高级-->属性

# sqlserver

vs自带简易的sql server服务器,去除了一些管理器等。连接的方法可以在资源管理器右键-->编辑连接-->高级里查看到。
连接方式为读取mdf文件绑定到sqlserver实例.
```bash
data source = addr;AttachDbFileName = file path;interact link=true;
# 实例地址         绑定文件名                   可信连接
```

注：复原实例要下载数据库管理工具ssms，出现mdf文件版本不匹配时,在vs中，新建连接选择mdf文件，出现提示框点击确定，会自动转化。

# iis身份认证

在iis应用程序中可以设置身份认证方式
若不存在基本身份认证，则需要在打开或关闭功能中添加基本身份认证
身份认证的账号密码似乎是服务器电脑的系统用户名密码，不可是别名
用户名一般是administrator

# asp.net form身份认证

form身份认证是需要数据库的.
当你安了sql server的时候，由于登陆默认采用的是本地sql server实例,，且程序会自动替你创建所需数据库,采用form登陆时即使没有建立合适的数据库来存放用户数据,也不会出现问题。
但若是没有安装sqlserver而是直接采用了vs自带的小型数据库，应用找不到默认的实例，便会报错。需要在配置文件中手动配置修改记录用户信息要用到的数据库。
处理方式如下：
1.  找到vs中的实例，记住实例名
2.  安装合适的数据库表，用命令如下
    ```bash
    #具体操作参数请自行在命令行查看，以下命令代表安装所有需要的服务
    aspnet_regsql -S 实例名 -E -A all
    ```
3.  修改web.config
    ```xml
    <!-- 添加内容如下 -->
    <connectionStrings>
    	<add name="ASPNETDB" connectionString="你的连接字符串"   providerName="system.data.sqlClient"/>
    </connectionStrings>
    <membership defaultProvider="SqlProvider"
      userIsOnlineTimeWindow="15">
      <providers>
        <add
          name="SqlProvider"
          type="System.Web.Security.SqlMembershipProvider"
          connectionStringName="ASPNETDB"
          applicationName="MyApplication"
          enablePasswordRetrieval="false"
          enablePasswordReset="true"
          requiresQuestionAndAnswer="true"
          requiresUniqueEmail="false"
          passwordFormat="Hashed"
          maxInvalidPasswordAttempts="5"
          passwordAttemptWindow="10" />
      </providers>
    </membership>
    ```
4.  [官网说明](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.sqlmembershipprovider?view=netframework-4.7.2)
