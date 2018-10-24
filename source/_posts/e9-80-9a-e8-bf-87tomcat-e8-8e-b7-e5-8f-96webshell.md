---
title: 通过Tomcat获取Webshell
tags:
  - Tomcat
  - Webshell
  - 服务器配置
id: 270
categories:
  - Webshell
date: 2016-10-24 23:14:57
---

## 导语：

在本地搭建了一个Tomcat服务器，尝试了一下利用，Tomcat manager中war包部署获取目标网站的Webshell。又了用了kali下的msf生成的war包进行shell的反弹。<!--more-->

<!--more-->

## 正文：

### 0x01.Tomcat环境部署

首先进行的是部署Tomcat的环境，安装Tomcat之前，必须先安装好JDK并配置好环境变量才能正常的运行Tomcat。

利用包管理工具安装了JDK 6 ，在/etc/environment中配置环境变量：

CLASSPATH=/usr/lib/jvm/java-6-openjdk-amd64/lib
JAVA_HOME=/usr/lib/jvm/java-6-openjdk-amd64

配置成功后，可以尝试使用java和javac命令确认是否配置好

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/3e8f2a6f018d1f6e8e0761e7fcb294d5.png)

JDK配置完成后，可以开始部署Tomcat服务器了。

<span style="font-size: small;">$sudo wget wget </span>[<span style="font-size: small;">http://archive.apache.org/dist/tomcat/tomcat-5/v5.5.20/bin/apache-tomcat-5.5.20.tar.gz</span>](http://archive.apache.org/dist/tomcat/tomcat-5/v5.5.20/bin/apache-tomcat-5.5.20.tar.gz)

执行命令获得Tomcat的压缩包，解压后，将文件移动到了/opt/目录下，进入到Tomcat/bin目录下，启动Tomcat：

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/8410b39b895e1f64f793dab52b3abb18.png)

出现这样的显示时，说明Tomcat启动成功。

访问Tomcat根目录，http://localhost:8080

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/8cd44b58f6e1459bda2f058eb5fce39a.png)

### 0x02.通过war包部署拿shell

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/fc6939efdbb54c8329bffa60d8219229.png)

可以看到页面左上角的Tomcat Manager,点击进入的时候，会出现http基础认证。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/fc4442058247d84ca6651ec95b7e28b5.png)

不知道账号密码时，可以在Tomcat根目录下的/conf/tomcat-users.xml。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/1d4ac68b4dad0fffac493f4d23b4004f.png)

rolename表示用户/权限，username表示账号，password密码，roles指顶用户的权限，可以有多个权限必须逗号分隔。Http的基础认证，输入过一次账号密码就无法登出了，必须关闭浏览器或者清除浏览器缓存来退出。（ps:manager是管理员权限）

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/44e1aff2b85382914efff0afd01e9da5.png)

进入管理页面后，可以进行war包部署了，从网上找到一个带有大马的war包进行部署，部署成功后，可以直接进入

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/7ee69c14837f6797d2cd65bd5f21bfde.png)

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/08d6ceb03bc62dc9cd8ef511fbc5c85c.png)

成功连接大马，尝试执行命令发现是root用户。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/fa1388f39e04939588f5e90f2f4a400b.png)

这是因为之前我在启动Tomcat服务器的时候，使用的是root用户启动的，当换成普通用户启动的时候，权限自然编程普通权限。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/a2c82a5dbeec74a8d0c7216e27a81f76.png)

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/1bc3c2d803dae616c3174286c86a1645.png)

所以说我们在部署网站的时候，尽量使用低权限的用户进行部署，提高整个站点的安全性，否则网站被拿webshell后直接就是root权限，无需提权。

这里还可以使用自己的一句话木马覆盖war包中自带的木马，但是不能通过普通的压缩软件来制作war包，虽然能够压缩，但是上传后无法正常部署。

### 0x03.利用msf反弹shell

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/40efd5c1a0393a47f7d1cafc799b43e1.png)

先使用相应的模块生成一个含有shell的war包，填写好监听主机的IP和端口

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/b3d174c9f0d859a2ac5dfcbfa8e76f4a.png)

然后启用监听模块，配置好IP和端口，执行等待shell反弹，将刚刚生成的war包像之前一样部署到Tomcat的根目录，然后访问部署的文件。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/274f2c3a69bde973f36b07fe2d9c0220.png)![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/aca36875e32416e9cb20a2143fb0d5a0.png)

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/10/6a63ae7fd022871fe3f271fdad47497c.png)

成功反弹回shell

## 总结：

Tomcat利用这种方式反弹shell的关键是，war包的部署，我们可以通过关闭Tomcat manager功能，或者是降低登陆manager页面用户的权限等方法来防止漏洞的产生，还需要保管好manager的账户密码以防泄露，防止漏洞。