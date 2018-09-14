---
title: IIS+PHP+MySql环境搭建
tags:
  - PHP
  - Windows安全配置
  - 服务器
id: 195
categories:
  - 数据库
date: 2016-08-03 00:27:42
---

### 导语

今天尝试了一下在IIS下搭建PHP环境，中间遇到了不少问题，现在来总结一下
<!--more-->
#### 正文

搭建这一套环境我用了以下软件：

*   PHP 5.3.10
*   IIS 6.0
*   Mysql 5.6.24
*   vcredist_x86.exe
*   fcgisetup_1.5_x86_rtw.msi（很重要）

大概先捋一捋这几个软件的关系，通过我自己的配置，我发现这些软件在配置的过程当中是这样联系在一起的。首先搭建的是IIS 6.0服务器，然后我们安装fcgisetup，这个模块是为了能让IIS支持PHP，IIS加载了FastCGI这个模块，从而有了PHP解析的功能，PHP中开启了Mysql和phpmyadmin的支持模块，从而将整个Web系统构架而成，现在我们来说说具体的配置方法。

#### IIS配置

1.进入控制面板&gt;&gt;程序和功能&gt;&gt;启用或关闭Windows 功能

2.找到Internet信息服务，安装IIS服务

3.安装完成后测试IIS是否安装成功，在浏览器中输入http://localhost将会出现下面的界面，否则安装不成功。

#### 安装配置PHP环境

1.下载并解压PHP到任意的目录下，比如：D：\PHP；

2.配置PHP，将D：\PHP\php.ini-development改名为php.ini：

*   修改时区date.timezone="Asia/Shanghai",去掉前面的分号；
*   激活需要的扩展php_gd2.dll php_mbstring.dll php_mysql.dll php_mysqli.dll php_pdo_mysql.dll
*   设置扩展路径extension_dir="D:\PHP\ext"
*   将php.ini保存到C：\windows下

#### 配置IIS支持PHP

1.安装fastcgi，安装完成后，IIS6里多了一个FastCGI Handler的

2.配置fastcgi，打开C:\WINDOWS\system32\inetsrv\fcgiext.ini

> 在最后添加了Exepath=C:\PHP\php-cgi.exe

3.增加扩展名 右键网站 =》 属性 =》 主目录 =》 配置 =》 添加，如下图配置： 可执行文件路径：C:\WINDOWS\system32\inetsrv\fcgiext.dll 扩展名填写.php 动作-&gt;限制为GET,HEAD,POST

#### 安装Mysql

1.  Mysql安装文件分为msi格式和zip格式两种，使用msi格式安装的时候，系统会自动配好Mysql，并打开服务，使用zip方式安装mysql需要我们手动对文件进行配置
2.  解压mysql到一个目录下。例如C:\Mysql
3.  配置mysql的环境变量，C:\Mysql\bin
4.  修改mysql的配置文件，打开配置文件mysql.ini将basedir（Mysql所在目录）和datadir（mysql所在目录）这两项配置正确
5.  运行CMD，进入C：\Mysql\bin目录下，输入命令，mysqld -install，显示安装成功后，输入命令net start mysql启动mysql服务
6.  输入mysqladmin -u root password “你的密码”，设置好root的密码
> 完成了以上配置以后，整个环境就搭建完成了，我们可以在网站的根目录放一个phpinfo（）的探针，测试各项是否配置成功，在查看一下Mysql等扩展组件是否成功打开。

#### 总结

整个配置的过程中需要注意的是，在配置文件的过程当中，一定要注意文件的路径是否填写正确，并且每配置完一个组件时就测试一下这个组件是否安装成功，不然一次安完后不容易发现问题出在哪。 通过配置这种非集成环境的时候，可以加强我们对环境中各个组件的理解，他们是通过什么样的方式集成在了一起，有助于我们对整个应用架构的理解和掌柜。

补充一下自己这两天搞wordpress全站静态化的经验，我们如果要将网站静态化，也就是将PHP解析成html文件，来优化我们的网站速度，可以使用Really Static和wp super cache这两个插件，这里说说两个插件的注意事项。

*   如果使用Really Static的话，需要注意两个事项：

1.  首先保证自己静态文件储存的目录和我们网站的链接指向的是同一个目录，及在第二步设置时要填写的那两个路径，一般选择网站根目录；
2.  第二要保证的时，你填写的目录必须有读写权限，这样才能保证第三步中服务器检测没有问题

*   如果使用wp super cache插件的话，我们只需要配置好插件的设置，选择全站缓存就好了，他就会自动将整站缓存为html文件，保存在指定的目录里，需要注意的是，最好我们的URL有伪静态并且我们的服务器是Apache或者IIS 7.0以上，保证我们具有mod_rewrite这个模块，我们才能顺利的将文章缓存，不然会出现错误。
&nbsp;

* * *

补充一下，今天我有搭建了IIS 7.5加PHP5.5的环境，需要注意的是安装过程中当PHP是基于VC11进行开发的时候，需要安装对应版本的vc++的环境，否则PHP解析不成功会爆出500错误。

**总结一下，如果出现错误，可能出现的情况有三种：**

1.PHP配置文件出错，或者是安装的vc运行环境出错；

2.文件的权限出现了问题，检查PHP用户的权限，以及运行程序相应文件夹的权限问题；

3.安装的应用程序有问题。

&nbsp;