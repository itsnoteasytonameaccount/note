---
title: catalina配置文件
date: 2018/12/12
tags:
- catalina
---

# 服务器禁止远程访问
<!-- more -->
[这个](https://blog.csdn.net/laokaizzz/article/details/78423216)

在对应tomcat的 conf/Catalina/localhost/ 文件夹下 增加一个manager.xml ，里面写上 
```xml
<Context privileged="true" antiResourceLocking="false" docBase="${catalina.home}/webapps/manager">
	<Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="127\.0\.0\.1" />
</Context>
```