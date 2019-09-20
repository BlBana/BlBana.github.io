---
title: Python安全编码——SQL注入
categories:
  - Web安全
tags:
  - SQL注入
  - Python
toc: true 文章目录
author: BlBana
date: 2019-09-20 18:10:56
description:
comments:
original:
permalink:
---

> 好久也没写过博客了，最近审计做的比较多，打算开个新坑，主要还是安全编码的问题，经常开发让修洞的时候，反问一句怎么修 ？乍得这么一问，还真有点一脸懵逼，还是收集一下常见的漏洞修复方案，以及一些安全SDK吧，今天讲讲Python SQL注入问题，都9012了还有人拼接SQL。。。是ORM不好用吗，还是SQL学的太好了。。

<!-- more -->

---

# 0x01 Python SQL安全编码

## 0x01-1 ORM框架

最好还是用ORM框架之类的执行SQL，现在Python开发Django用的也比较多，Django自带的ORM真的好用啊，model一写，用起来舒舒服服还不用担心注入。**推荐，首选ORM。**

## 0x01-2 参数化查询

Python的`cursor.execute`方法已经自带了参数化查询方式，使用方法如下：

```python
username = "admin"
password = "pass"
sql = "select * from user_table where username = %s and password = %s"
args = (username, password)
cursor.execute(sql, args)  # python查询方法已经提供了参数化查询的方式，自动处理过滤
```

> 记住用的时候，用" , "不要用" % "

## 0x01-3 过滤参数

过滤参数的话主要还是`\x00, \n, \r, \, ', " 和 \x1a`几个字符为主，要么根据业务需求把参数类型卡死一点或者正则匹着来，后面打算写个Python的安全SDK开源，自己先占个坑，Python MySQLdb也自带了过滤的方法。

- `MySQLdb.escape_string`实现了MySQL C接口的`mysql_escape_string`主要就是过滤了上述的几个字符



# 0x02 总结

注入感觉现在见的也不多了，各种框架，ORM都帮我们处理好了，正确的使用这些工具其实已经能避免SQL注入了，再加上现在各类防护产品，注入真心少见了。通用注入修复思路和安全编码：

- 使用SQL预编译和存储过程对参数进行绑定**（推荐）**
- 动态拼接语句需要使用安全API对参数进行过滤
- 根据业务需求严格控制传入参数类型以及形式
- 使用ORM框架查询，框架内基本都对SQL注入有处理

> 没了，溜了。。感觉自己好水。。我要回炉重造了，886