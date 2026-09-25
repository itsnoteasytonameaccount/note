---
title: oracle数据库操作
tags: oracle
date: 2018/05/16
---

# 前言
感觉oracle使用时较mysql麻烦不少，记录一下学到的操作方法
<!--more-->

# 操作
1.  获取实例名
```bash
select instance_name from V$instance;
```
2.  获取数据库名
```bash
select name from V$database;
```
3.  创建表空间
```bash
create tablespace 表空间名
logging
datafile 文件路径
size 100m
autoextend on
next 10m
maxsize unlimited
extent management local
```
其中logging代表创建重做日志，具体意思目前不明，可不填
extent management local代表创建本地管理表空间，具体意义不明，可不填
datafile path代表表空间文件存放路径
size文件大小
autoextend on开启自动扩展空间功能
maxsize unlimited最大大小不限
next 10m扩展大小10m
4.  删除表空间
```bash
drop tablespace 表空间名 including contents and datafiles;
```
contents表示内容，表空间不为空时必须添加。
datafiles表示表空间文件。

5.  获取表空间信息
```bash
select 信息名 from dba_tablespaces;
```

6. 创建表
```bash
create table 表名(变量名 类型,变量名 类型);
```
可选参数

|参数|作用|
|:-:|:-|
|primary key|设为主键|
|unique|不可重复|
|not null|不可为空|
|default 'value'|默认值为value|
|references table(name)|设为对应table表中name字段的外键|
|tablespace name|指定表创建在哪个表空间下，若不选，则创建在用户默认表空间下|

例子
```bash
create table mytable(
id varchar(10) not null primary key,
describe varchar(20),
state number(1) not null default 1
) tablespace mytablespace;
```

7.  获取表信息
```bash
select 信息名 from dba_tables;
```

8.  删除表
```bash
drop table 表名
```

9.  修改表字段信息
```bash
alter table 表名 要做的操作
//例如，添加外键依赖
alter table mytable
add constraint your_fk_name foreign key(name)
references an_other_table(an_other_name);
//添加主键依赖
later table mytable
add constraint you_pk_name primary key(name);
```

10.  获取用户信息
```bash
select 用户信息 from dba_users;
```

11.  获取表的字段名
```bash
select t.column_name from user_tab_columns t where t.tale_name='表名';
```

12. 创建用户并指定表空间
```bash
create user UserName identified by Password
default tablespace t
temporary tablespace t;
-- 指定
alter user UserName temporary tablespace t;
```
13. 表空间里存放了哪些用户的数据：
```bash
select distinct owner,tablespace_name from dba_segments;
```
