---
title: Mysql宽字节注入
categories:
  - SQL注入
tags:
  - SQL注入
  - 宽字节注入
toc: true 文章目录
author: BlBana
date: 2016-12-05 20:12:35
description:
comments:
original:
permalink:
---
>现在大多数的网站对于SQL注入都做了一定的方法，例如使用一些Mysql中转义的函数addslashes，mysql_real_escape_string，mysql_escape_string等，还有一种是配置magic_quote_gpc，不过PHP高版本已经移除此功能。其实这些函数就是为了过滤用户输入的一些数据，对特殊的字符加上反斜杠“\”进行转义，这里讨论一下利用宽字节注入绕过这些函数。
<!-- more -->

## 字符集简介
GBK是一种多字符的编码，通常来说，一个gbk编码汉字，占用2个字节。一个utf-8编码的汉字，占用3个字节。当将页面编码保存为gbk时输出2，utf-8时输出3。除了gbk以外，所有ANSI编码都是2个字节。现在我们来研究一下SQL注入中，字符编码带来的问题。

---
## 开始测试

### 0x01 Mysql中宽字节注入

实例代码如下：
```php
  <?php
  //连接数据库部分，注意使用了gbk编码，把数据库信息填写进去
  $conn = mysql_connect('localhost', 'root', '07191226') or die('bad!');
  mysql_query("SET NAMES 'gbk'");
  mysql_select_db('test', $conn) OR die("连接数据库失败，未找到您填写的数据库");
  //执行sql语句
  $id = isset($_GET['id']) ? addslashes($_GET['id']) : 1;
  $sql = "SELECT * FROM news WHERE tid='{$id}'";
  $result = mysql_query($sql, $conn) or die(mysql_error()); //sql出错会报错，方便观察
  ?>
  <!DOCTYPE html>
  <html>
  <head>
  <meta charset="gbk" />
  <title>新闻</title>  
  </head>
  <body>
  <?php
  $row = mysql_fetch_array($result, MYSQL_ASSOC);
  echo "<h2>{$row['title']}</h2><p>{$row['content']}<p>\n";
  mysql_free_result($result);
  ?>
  </body>
  </html>
```
在这个语句中，使用了addslashes函数，将我们输入的$id进行了转义。
现在我们对这个脚本进行SQL注入测试，先加上一个单引号：
![|center](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/MySql-sqli/1.jpg)

正常的显示，没有报错，说明addslashes起到了作用，我们的单引号被转义了，没有起到作用。这个时候如果想要逃过addslashes的限制，我们有两种办法：

>1.想办法让\前再加一个\，这样就成了\\'，\被转义，‘逃出了限制；
>2.先办法把\弄没有。

这里就可以利用宽字节注入来进行绕过，mysql在使用GBK编码的时候，会认为两个字符是一个汉字，当我们输入%df的时候：
![|center](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/MySql-sqli/2.jpg)

发现出现了报错，说明我们的语句已经影响了正常语句的执行了，可以注入了。这里为什么加一个%df就可以了呢？因为这是mysql的一种特性，GBK是多字节编码，它认为两个字节就代表一个汉字，在%df加入的时候会和转义符\，即%5c进行结合，变成了一个“運”，而’逃逸了出来。

因此只要第一个字节和%5c结合是一个汉字，就可以成功绕过了，当第一个字节的**ascii码大于128**，就可以了。

### 0x02 GB2312测试
gb2312和GBK一样属于宽字节，我们尝试一下gb2312能否绕过。
```php
<?php
  //连接数据库部分，注意使用了gbk编码，把数据库信息填写进去
  $conn = mysql_connect('localhost', 'root', '07191226') or die('bad!');
  mysql_query("SET NAMES 'gb2312'");
  mysql_select_db('test', $conn) OR die("连接数据库失败，未找到您填写的数据库");
  //执行sql语句
  $id = isset($_GET['id']) ? addslashes($_GET['id']) : 1;
  $sql = "SELECT * FROM news WHERE tid='{$id}'";
  $result = mysql_query($sql, $conn) or die(mysql_error()); //sql出错会报错，方便观察
  ?>
```
测试发现已经不可以注入了：

![|center](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/MySql-sqli/3.jpg)

这是因为gb2312的取值范围。高位的范围是0xA1~0xF7，低位范围是0xA1~0xFE，\是0x5C，不在地位的范围中。所以在编码中，’不会被吃掉，就无法造成注入了。

### 0x03 测试mysql_real_escape_string
mysql_real_escape_string，函数在使用的时候一定要考虑到**当前字符集（mysql_set_charset）**
现在将源码中的addslashes替换成mysql_real_escape_string，来尝试一下。
```php
 <?php
  //连接数据库部分，注意使用了gbk编码，把数据库信息填写进去
  $conn = mysql_connect('localhost', 'root', '07191226') or die('bad!');
  mysql_query("SET NAMES 'gb2312'");
  mysql_select_db('test', $conn) OR die("连接数据库失败，未找到您填写的数据库");
  //执行sql语句
  $id = isset($_GET['id']) ? mysql_real_escape_string($_GET['id']) : 1;
  $sql = "SELECT * FROM news WHERE tid='{$id}'";
  $result = mysql_query($sql, $conn) or die(mysql_error()); //sql出错会报错，方便观察
  ?>
```
![|center](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/MySql-sqli/4.jpg)
同样可以造成注入，这主要是因为没有指定php连接mysql时的字符集。我们需要在执行SQL语句前加上一个mysql_set_charset函数，设置当前字符集为gbk。
`
mysql_set_charset('gbk' , $conn);
`
这样子就可以避免注入了。

### 0x04 防止宽字节注入
一种方法就是先调用`mysql_set_charset`设置当前字符集为gbk，再调用函数mysql_real_escape_string来过滤用户输入。

还可以把`character_set_client`设置为binary二进制

当我们的mysql接受到客户端的数据后，会认为他的编码是character_set_client，然后会将之将换成character_set_connection的编码，然后进入具体表和字段后，再转换成字段对应的编码。然后，当查询结果产生后，会从表和字段的编码，转换成character_set_results编码，返回给客户端。

### 0x05 iconv导致注入
使用函数`iconv('utf-8','gbk',$_GET['id'])`,也可能导致注入产生。
虽然`character_set_client`设置成了binary，但是之后参数又被变回了gbk，我们可以使用“ 錦' ”来进行注入：
![|center](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/MySql-sqli/5.jpg)
这是因为錦'的GBK编码为0xe55c，所以就能发现后面为%5c%27，这个时候会出现两个%5c，刚好\\，反斜杠被转义，导致‘逃出造成了注入

### 0x06 总结
这里也发现，我们在编程的过程中尽量使用UTF-8的编码，从而避免宽字节的注入。即使使用GBK编码，也一定要做好防范：

>1.设置character_set_client=binary；
>2.单独的使用set names gbk或者mysql_real_escape_string时也需要使用mysql_set_charset来设置一下字符集；
>3.尽量不要使用iconv来转换字符编码。

