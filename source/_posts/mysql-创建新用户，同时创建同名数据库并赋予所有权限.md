---
title: mysql 创建新用户，同时创建同名数据库并赋予所有权限
date: 2024-03-16 14:26:45
tags:
  - mysql
  - sql
categories:
  - mysql
---

服务器新建站点时，经常会要求新建一个数据库帐号，并新建与帐号同名的数据库，并赋予所有权限，在此记录下使用的代码。

```shell
[root@ecs-2c24 ~]# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 251978
Server version: 5.6.43-log MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
// 创建名为 zbsc 的数据库
MySQL [(none)]> create database zbsc;
Query OK, 1 row affected (0.00 sec)
// 创建名为zbsc的用户，允许从任何IP以密码方式登录
MySQL [(none)]> create user 'zbsc'@'%' identified by '密码';
Query OK, 0 rows affected (0.00 sec)
// 赋予zbsc用户对zbsc数据库下所有表的完整权限
MySQL [(none)]> grant all on zbsc.* to 'zbsc'@'%';
// 赋予zbsc用户对zbsc数据库下所有表的只读权限
MySQL [(none)]> grant select on zbsc.* to 'zbsc'@'%';
Query OK, 0 rows affected (0.00 sec)
// 刷新权限表，使其立刻生效
MySQL [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)
// 收工
```