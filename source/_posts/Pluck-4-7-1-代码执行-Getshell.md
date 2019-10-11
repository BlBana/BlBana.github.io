---
title: Pluck 4.7.1 代码执行 Getshell
categories:
  - Web安全
tags:
  - Web安全
  - 漏洞分析
toc: true 文章目录
author: BlBana
date: 2019-10-11 16:30:23
description:
comments:
original:
permalink:
---

> 好久没有接触过PHP了，有的好多东西都记得不是太清楚了，最近打算找一些CMS的漏洞分析分析，把PHP再拾起来。今天刚好看见先知有人发[相关文章](https://xz.aliyun.com/t/6486)，就跟着分析调试了下。

<!-- more -->

---

# 漏洞描述

Pluck是个小型的CMS系统，整个代码量也比较少，整个系统没有用到数据库，后台编写文章以后，会将用户输入的文章标题，内容等信息，动态的拼接成PHP变量声明代码，并写入到PHP文件当中，当浏览文章时，通过PHP文件包含将数据文件包含进来，从变量中获取文章信息。由于部分字段没有过滤，导致写入任意代码，包含时造成代码执行。

漏洞版本：4.7.1

# 漏洞分析

漏洞入口文件在`admin.php`中：

当`action`变量值为`editpage`时，包含`editpage.php`文件

![](	https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Pluck-Getshell/start.png)

跟进`editpage.php` 文件：

![](	https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Pluck-Getshell/editpage.png)在保存和修改文章内容时都会调用`save_page`函数，文章`title`，`content`，`hidden`等都来自用户输入。

继续跟进`save_page`函数，其中会根据文章`title`生成要保存数据的PHP文件路径`newfile`变量，之后开始处理输入变量，并拼接PHP文件内容，title和content变量都使用sanitize进行过滤处理。

![](	https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Pluck-Getshell/data.png)

跟进sanitize函数：

```php
function sanitize($var, $html = true) {
    $var = str_replace('\\', '\\\\', $var);
    $var = str_replace('\'', '\\\'', $var);

    if ($html == true)
        $var = htmlspecialchars($var, ENT_COMPAT, 'UTF-8', false);

    return $var;
}
```

sanitize会对单引号，反斜杠进行转义处理，并对HTML内容进行实体编码。

![](	https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Pluck-Getshell/save_file.png)

处理结束后调用save_file函数将data变量保存到newfile中。

![](	https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Pluck-Getshell/fputs.png)

## PoC构造

PoC主要是通过拼接`data`的几个变量来进行构造，`data`变量最终如下：

```php
<?php
	$title = 'title';
	$content = 'content';
	$hidden = 'hidden';
?>
```

构造PoC的关键是闭合单引号，使我们输入的数据能被当做代码执行，但是`title`，`content`变量已经被过滤，无法闭合单引号，因此通过`hidden`变量来写shell，具体PoC如下：

```php
test';phpinfo();//
```

PoC构造需要注意两个分号，避免语法错误。

# 漏洞复现

在请求 `title，content，hidden`三个传入PoC，写入文件

![](	https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Pluck-Getshell/PoC.png)

进入editpage.php和index.php页面都会触发代码执行

![](	https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Pluck-Getshell/phpinfo.png)

漏洞触发位置

- `editpage.php —> 22行 require_once 包含PHP文件`
- `index.php —> 99行 include_once('default/theme.php') —> theme.php 12行 theme_menu() —> functions.site.php 168行 theme_menu_data() —> functions.site.php 191行 include 包含PHP文件`

# 漏洞修复

对`hidden`变量进行过滤，防止闭合单引号，构建PHP代码。

- `escapeshellarg` —— 将给字符串增加一个单引号并且能引用或者转码任何已经存在的单引号

在最新版本4.7.10「2019.8.1」中的`sanitize`过滤更加严格，但依然没有给hidden使用该函数，还是存在写入shell的风险。