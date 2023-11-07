---
title: mysql记录
categories: Kubernetes
sage: false
date: 2019-9-19 14:48:18
tags: mysql
---

## 添加一个字段

ALTER TABLE jw_user_role ADD zk_env VARCHAR(16);

## 修改字段为not null，还要把原来的类型也写出来

ALTER TABLE jw_user_role MODIFY  zk_env VARCHAR(16) NOT NULL;

<!-- more -->

## 更改列名

alter table student change physics physisc char(10) not null;

## 先来看看常用的方法

MySql的简单语法，常用，却不容易记住。当然，这些Sql语法在各数据库中基本通用。下面列出：

1.增加一个字段

alter table user add COLUMN new1 VARCHAR(20) DEFAULT NULL; //增加一个字段，默认为空
alter table user add COLUMN new2 VARCHAR(20) NOT NULL; 　　 //增加一个字段，默认不能为空

2.删除一个字段

alter table user DROP COLUMN new2; 　　　　　　　　　　　　　　 //删除一个字段

3.修改一个字段

alter table user MODIFY new1 VARCHAR(10); 　　　　　　　　　　 //修改一个字段的类型
alter table user CHANGE new1 new4 int;　　　　　　　　　　　　　 //修改一个字段的名称，此时一定要重新

## 主键

alter table tabelname add new_field_id int(5) unsigned default 0 not null auto_increment ,add primary key (new_field_id);

## 增加一个新列

alter table t2 add d timestamp;
alter table infos add ex tinyint not null default ‘0′;

## 删除列

alter table t2 drop column c;

## 重命名列

alter table t1 change a b integer;

## 改变列的类型

alter table t1 change b b bigint not null;
alter table infos change list list tinyint not null default ‘0′;

## 重命名表

alter table t1 rename t2;

## 加索引

mysql> alter table tablename change depno depno int(5) not null;
mysql> alter table tablename add index 索引名 (字段名1[，字段名2 …]);
mysql> alter table tablename add index emp_name (name);

## 加主关键字的索引

mysql> alter table tablename add primary key(id);

## 加唯一限制条件的索引

mysql> alter table tablename add unique emp_name2(cardnumber);

## 删除某个索引

mysql> alter table tablename drop index emp_name;

## 增加字段

mysql> ALTER TABLE table_name ADD field_name field_type;

## 修改原字段名称及类型

mysql> ALTER TABLE table_name CHANGE old_field_name new_field_name field_type;

## 删除字段

mysql> ALTER TABLE table_name DROP field_name;

## mysql修改字段长度

alter table 表名 modify column 字段名 类型;

例如

数据库中user表 name字段是varchar(30)

可以用

alter table user modify column name varchar(50);
