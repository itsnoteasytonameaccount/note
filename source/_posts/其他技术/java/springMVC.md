---
title: 关于springBoot
date: 2018/10/3 12:00:00
tags:
- java
---

因为有用，接触了一下springboot记录成果。

# servlet

### Servlet接口
Servlet接口内容如下
```java
    public interface Servlet {
        void init(ServletConfig var1) throws ServletException;
        ServletConfig getServletConfig();
        void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;
        String getServletInfo();
        void destroy();
    }
```
### init函数
请求到达时，检测相应Servlet对象是否存在，不存在调用init方法初始化，init在生命周期中，只调用一次

### servlet处理方式

单实例多线程，实例创建一次，请求到来时新建线程处理会话


# springMVC
[别人的好文章](https://blog.csdn.net/u011514729/article/details/79689637)
springmvc的处理方式大概是用一个sevlet类来处理所有请求，将这些请求分发给用户定义的类处理（仅本人理解，无可靠依据）
[load-on-startup作用](https://blog.csdn.net/u012393192/article/details/79323979)
```xml
<!--配置分发请求用的sevlet-->
    <servlet>
        <servlet-name>springMVC</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!--bean配置，用户定义的处理函数-->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/conf/springMVC.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
<!--配置路径匹配-->
    <servlet-mapping>
        <servlet-name>springMVC</servlet-name>
        <!--精确匹配优先，长匹配优先-->
        <url-pattern>/springMVC/*</url-pattern>
    </servlet-mapping>
```

# springBoot

controller里接口参数对象若不加注解，默认接收post的multilpart/form-data格式数据，get的url里的query参数
@RequestBody注解整体接受application/json格式数据,只能是对象
@RequestParam可以接收form-data,x-www-form-urlencoded格式的，但只能是键值对，
@ReuqestAttribute接收request属性，转发，或过滤器处理过程中产生的
@RequestPart暂不明
不加注解可以处理form-data,x-www-form-urlencoded，可以是参数列表，也可以是对象，对象必须有标准getter,setter,和无参构造函数

## 可能碰到的问题

1. 启动类只会扫描同级目录下，或同级子目录下的类，因此若controller的包与启动类同级无法被检测到
2. 8.0以后版本的mysql驱动会有时间格式的冲突，需要在路径后指定时间格式，同时驱动包名也改了，如下
```bash
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/yourdatabase?serverTimezone=GMT
```
3. JpaRepository的@query中引用对象时必须要@param注解，更新操作需要@Modifying@Trasactional注解
4. ideal的maven选项中reimport似乎不会下载未下载的依赖，要用downloadresource(用的少不知道为什么)
5. 用select last_insert_id()获取最后插入id;
6. 更新操作应加上@Modifying,和@Transactional注解
7. 若字段与保留字冲突，使用反引号区分
8. 调用save函数需要通过generatedValue注解来指定主键生成方式
    a.  TABLE：使用一个特定的数据库表格来保存主键。 
    b.  SEQUENCE：根据底层数据库的序列来生成主键，条件是数据库支持序列。 
    c.  IDENTITY：主键由数据库自动生成（主要是自动增长型） 
    d.  AUTO：主键由程序控制。 
9. 可以用EntityManager自定义数据库查询方式，此时若是插入等操作，仍需要@Transactional注解
10. 嵌套的transactional注解会报错
11. transactional注解的函数中,catch语句无效，捕捉不到真正错误，因为@transactional将操作原子化，出错会回退，报roll-back-only错误