---
title: LAMP服务器的安全配置
id: 190
categories:
  - 未分类
tags:
---

## 导语

当我们在Linux上要搭建网站服务时，我们需要一个Apache服务器软件。搭配上Mysql和PHP我们就可以搭建一个动态的网站，但是安全的配置就显的十分重要了，配置错误会导致数据泄露，对我们造成危害

### LAMP所需软件

*   httpd
*   PHP
*   Mysql（客户端程序）
*   Mysql-Server（服务器端程序）
*   PHP-Mysql（提供PHP程序读取Mysql数据库的模块）

### Apache主要配置文件

*   /etc/httpd/conf/httpd.conf(主要配置文件)
*   /etc/httpd/conf.d/*.conf（额外参数文件）
*   /var/www/html（网站根目录）
*   /var/log/httpd（Apahce日志文件都在这里）

### Mysql主要配置文件

*   /etc/my.cnf（Mysql的配置文件）
*   /var/lib/mysql（Mysql数据库文件存储的地方）

### PHP主要配置文件

*   /etc/httpd/conf.d/php.conf
*   /etc/php.ini（PHP主要配置文件）

### Apache主要配置解析（httpd.conf）

*   ServerRoot "/etc/httpd"
> 服务器最顶层的目录，像logs，moudles等模块都应该在这个目录下存放

*   Listen 80
> 监听的端口

*   ServerAdmin [xxx@123.com](mailto:xxx@123.com)
> 系统管理员的Email，当网站出现问题的时候，错误信息会发到邮箱

*   ServerName [www.xxx.com](http://www.xxx.com)
> 设置主机名，主机名需要在hosts中进行与IP的指定

#### 网站首页及目录权限

*   DocumentRoot "/var/www/html"
> 指你放置首页的目录

    <span class="hljs-tag">&lt;Directory /&gt;</span>
            <span class="hljs-keyword"><span class="hljs-common">Options</span></span> FollowSymLinks
            <span class="hljs-keyword">AllowOverride</span> None
        <span class="hljs-tag">&lt;/Diectory&gt;</span>`</pre>
    > 设置WWW服务器默认的环境
    <pre data-source-line="43">`<span class="hljs-tag">&lt;Directory "/var/www/html"&gt;</span>
            <span class="hljs-keyword"><span class="hljs-common">Options</span></span> Indexes FollowSymLinks
            <span class="hljs-keyword">AllowOverride</span> None
            <span class="hljs-keyword"><span class="hljs-common">Order</span></span> allow,deny
            <span class="hljs-keyword"><span class="hljs-common">Allow</span></span> from <span class="hljs-literal">all</span>
        <span class="hljs-tag">&lt;/Diectory&gt;</span>

> 这个是针对这个目录各项权限的设置，参数含义如下：

*   Options（目录参数）
> 这个设置值表示这个目录中能够让Apache进行的操作，主要参数有：

1.Indexes：如果再次目录下找不到首页文件，例如：index.html，就显示当前目录下所有的文件名。**如果配置不当会让别人，很容易的探索清楚你的目录结构，从而了解到你所使用的CMS等信息**

2.FollowSymLinks：可以使用链接文件

3.ExecCGI：让此目录具有CGI程序的权限。注意**不能让所有的目录都具有此权限**

4.AllowOverride（允许覆盖的参数）：表示是否允许额外的配置文件.htaccess来覆盖某些参数。

*   ALL：全部的权限可以覆盖
*   AuthConfig：仅有网页认证可覆盖
*   Indexes：仅允许Indexes方面覆盖
*   Limits：允许用户利用Allow，Deny与Order管理可浏览的权限
*   None：不可覆盖

5.Order，allow，deny（能否浏览的权限）：

*   deny，allow：以deny优先处理，但没有写入规则默认为allow
*   allow，deny：以allow为优先处理，但没有写入规则默认为deny
> DiectoryIndex index.html index.php
> 当我们仅输入目录时，会自动在这个目录下搜索首页文件，如果里面的文件全部存在的话，就按顺序进行搜索，第一个的优先级最高