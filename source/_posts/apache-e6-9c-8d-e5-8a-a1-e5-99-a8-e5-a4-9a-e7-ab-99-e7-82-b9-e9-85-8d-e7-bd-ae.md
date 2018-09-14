---
title: Apache服务器多站点配置
tags:
  - 服务器配置
  - 运维
id: 174
categories:
  - 服务器
date: 2016-07-30 18:26:56
---

> 当我们需要在同一个服务器中搭建多个站点时，此时我们就需要使用到Apache服务器虚拟主机的配置，接下来我们详细介绍一下。

<!--more-->
* * *

1.让Apache运行时加载虚拟主机模块打开，在目录conf/httpd.conf文件。

```
#LoadModule vhost_alias_module modules/mod_vhost_alias.so
#Include conf/extra/httpd-vhosts.conf
去掉#并保存
```

2.在同一文件中找到DocumentRoot和Diectory，改为站点目录的上一级（改目录是所有网站根目录的父目录）

3.完成配置后，打开Apache安装目录下的/conf/extra/httpd-vhosts.conf文件：

```
DocumentRoot是文件放置路径，ServerName是网站域名：
<VirtualHost*:80>
DocumentRoot"D:/Appserv/www/1"
ServerName www.xxx.com

<VirtualHost*:80>
DocumentRoot"D:/Appserv/www/2"
ServerName www.xxx2.com

```

4.重启Apache完成所有配置