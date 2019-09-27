---

title: PHPStorm远程调试环境搭建以及原理
categories:
  - Web安全
tags:
  - 代码审计
  - PHP
toc: true 文章目录
author: BlBana
date: 2019-09-27 11:18:44
description:
comments:
original:
permalink:
---

> 最近准备做一些PHP的漏洞分析，来拓展一下自己代码审计时的思路，俗话说“工欲善其事必先利其器”嘛，就把PHPStorm代码调试环境配置了一下，本地比较容易，主要还是记录一下在配置远程调试环境的一些要点，用于备忘，再简单介绍一下远程调试的原理。

<!-- more -->

---

PHP大家一般会使用打印变量值这样的基础方法进行调试：

|    名称    |   类型   |                           描述                            |
| :--------: | :------: | :-------------------------------------------------------: |
|    echo    | 语言结构 |             打印简单类型变量（如int，string）             |
|   print    | 语言结构 |             打印简单类型变量（如int，string）             |
|  print_r   |   函数   |   打印出复杂类型变量的值(如数组、对象）以列表的形式显示   |
|  var_dump  |   函数   |        判断多个变量的类型和长度，并输出变量的数值         |
| var_export |   函数   |  var_dump() 类似，不同的是其返回的表示是合法的 PHP 代码   |
|    dump    |   函数   | 大部分框架自带的调试函数,用于var_dump()友好(格式化)的输出 |

但打印的方式用来调试大型的项目就显得力不从心了，这时候可以选择使用PHPStorm进行断点调试，分析漏洞的数据流。PHP需要安装额外的扩展进行调试，主要有以下两种，本文主要讲Xdebug相关内容。

- Xdebug
- Zend Debugger

# 0x01 环境介绍

- 本地：macOS 10.14.5 + phpstrom 2019.2.1
- 远程：Windows10 + phpstudy8

# 0x02 Server配置

1. Server端使用phpstudy的集成环境，给`php`下载配置对应版本的`Xdebug`扩展，不知道版本的话可以使用[Xdebug官方工具检测并提供下载链接](https://xdebug.org/wizard)。

2. 配置`php.ini`文件：

   ```ini
   [xdebug]
   zend_extension="E:\Develop-Env\XAMPP\php\ext\php_xdebug-2.7.2-7.3-vc15-x86_64.dll"
   xdebug.idekey = PHPSTORM  # 必填，用于发起调试请求的Cookie
   xdebug.remote_enable = 1  # 必填，开启远程调试
   xdebug.remote_autostart = 0  # 选填，自动开启调试模式
   xdebug.remote_connect_back = 1  # 选填，自动回联请求IP进行调试 （多人调试）
   xdebug.remote_host = 127.0.0.1  # 选填，监听调试机器IP，和上面这个参数选择开启一个 （单人调试）
   xdebug.remote_port = 9000  # 必填，监听调试的IDE机器端口
   xdebug.remote_handler = dbgp  # 必填，通信协议
   xdebug.scream = 0
   xdebug.show_local_vars = 1  # 后面这两个我复制的...
   ```

3. 配置完后重启WebServer即可。

# 0x03 本地IDE配置

## 0x03-1 Debug配置

### 0x03-1.1 DEBUG Port

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_debug.png)

配置要监听的端口，这里配置与上面php.ini文件配置保持一致为9000端口

### 0x03-1.2 DBGp Proxy

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/Xdebug_dbgp.png)

- Host为Server端IP地址
- Port为IDE监听的调试端口
- IDE Key为Server端php.ini中xdebug.idekey = PHPSTORM，**用于之后启动调试的参数**

## 0x03-2 Servers配置

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_server.png)

- Host为Server端IP地址
- Port为Server端Web服务端口
- Debugger为Server端debug扩展

> 远程调试需要开启`user path mappings`，将远程环境项目与本地环境项目进行映射。

- File/Directory --》本地路径
- Absolute path the server --》 远程路径

# 0x04 调试

> 完成上述配置以后就可以开始进行代码调试了，远程调试有两种不同的调试方法，分别是浏览器插件发起调试请求或者PHPStorm debug发起调试请求。

## 0x04-1 PHPStorm debug发起调试请求

1. 需要进行`Run/Debug configurations`配置，创建一个`PHP Web Page`的运行配置

   - Server：选择3.2步骤中Server
   - Start URL：配置要调试的页面路径
   - Brower：浏览器

   ![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_config_web.png)

2. 准备就绪后，开启PHPStorm右上角的小电话，绿色时为开启9000端口的debug监听，点击小爬虫拉起浏览器开始调试代码。

   ![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_config_web_start.png)

3. 之后就可以打断点，正常进行调试了。

> 不过这样子调试还是有些麻烦的，每次需要配置`Start URL`参数来控制自己要调试的页面，PHPStorm通过加`XDEBUG_SESSION_START`参数发起GET请求，来开启调试，服务端Xdebug在接收到请求后，会与IDE的9000端口发起连接，开始调试。不过这样子配置还是有些麻烦，每次要么需要改动`Start URL`参数或者在自己想访问的页面URL后加上`XDEBUG_SESSION_START`参数来开启调试，安装浏览器插件就舒服多了 ~

## 0x04-2 浏览器插件发起调试

1. 安装浏览器插件**Xdebug Helper**
2. 配置好插件的**IDE Key为PHPSTORM**
3. 在PHPSTORM中**开启小电话**，并且在浏览器中**开启插件的DEBUG模式**，打好断点，直接开始访问想要调试的页面就能开启调试了

> 其实调试原理都是一样的，通过插件的主要是在Cookie中放了一个**XDEBUG_SESSION=PHPSTORM**发起了开启调试的请求，这里也可以使用Postman之类的调试插件来发起请求，把**XDEBUG_SESSION=PHPSTORM**带上就OK

## 0x04-3 坑 ！！

一定要保证远程环境和本地环境代码一致，否则调试过程会出现问题，我下的cms所有代码都在一行，本地代码用phpstorm格式化显示了，但远程代码还在一行里，我调试的时候本地一直显示在第一行，单步调试光标也不动，配置翻来覆去看都觉得没问题，最后才想起来我本地代码格式化过，远程还是在一行，mmp，折腾了一下午。。。

有个比较好的办法就是开启PHPStorm的远程开发功能，配置好远程开发环境**Deployment**，每次修改完本地代码后PHPStorm可以自动同步远程和本地代码，还能进行远程和本地的代码对比，代码上传/下载等功能。具体配置可以看第六部分。

# 0x05 原理分析

PHPStorm已经集成了遵循BGDP协议的Xdebug插件，当开启小电话的时候，会在本地开启一个Xdebug的调试服务，监听端口就是我们在上面配置的9000端口。

当浏览器送一个带有**XDEBUG_SESSION**参数的请求到服务器时，服务器会将其交给后端PHP处理，当开启了Xdebug功能时，就会把debug信息通过9000端口发送到客户端的IDE上，从而完成调试功能。开启调试的关键就是**XDEBUG_SESSION**参数，可以通过增删这个参数来控制是否开启调试功能。

> 有个要注意的点就是Xdebug的配置中有个参数**xdebug.remote_autostart = 0**，我看很多文章都配置为1，这个参数是指是否自动开启调试模式，当参数置为1时，不管你请求中是否带有**XDEBUG_SESSION**参数，都会进入调试模式，所以这里建议关掉。

这里附上Xdebug官网的IDE和Xdebug之间调试关系图。

1. 图1为开启xdebug.remote_host参数的单人调试模式，只会给指定IP的请求开启调试

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_xdebug_only.png)

2. 图2为开启xdebug.remote_connect_back参数的多人调试模式，会根据请求的IP信息进行回连

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_xdebug_multi.png)

# 0x06 远程开发

>  大概再介绍一下远程开发，有了这个功能，就能方便的在本地调试代码并实时修改代码并进行同步工作了。

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_remote_dev.png)

1. 如图在创建新项目时候选择加载已有的项目，并选择使用FTP来加载远程项目
2. 跟着步骤配置好本地路径和FTP服务器信息后就能开启远程开发模式了

> 这项配置可以选择是否进行自动代码同步

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_remote_dev_config.png)

右键点击想要同步的文件夹或者文件，点击Deployment就能选择上传/下载/对比/同步这几个功能，试了一下真的贼好用 ~ 舒服了 ...

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/phpstorm-debug/xdebug_remote_dev_upload.png)

# 0x07 总结

PHP调试环境配置起来还蛮麻烦的，不过跟着步骤一步步来配上一次就OK了，之后就是愉快的开始调试漏洞了 ~ 溜了溜了

> 菜鸡打滚 ~