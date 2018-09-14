---
title: Webshell提权（一）
tags:
  - Webshell
  - 提权
id: 80
categories:
  - Webshell
date: 2016-07-28 10:28:33
---

> 在我们拿到Webshell以后，无论是一句话小马还是使用大马，我们第一个想到的就是使用CMD命令给我们提权，创建用户并加入到管理员组。

<!--more-->
### 常用的提权方法

1.尝试直接使用Webshell执行CMD命令

1.  服务器上支持php，asp，jsp时，利用这些脚本的可执行系统命令的函数进行提权。
> PHP ：
> 
> 
> *   exec()函数原型string exec(string $cmd [,array &amp;$output[,int &amp;$return_var]])
> 
> 说明：exec执行系统外部命令后不会输出所有的结果，而是返回结果的最后一行，如果想得到所有的数，利用第二个参数让他输出到指定的数组，最后打印这个数组
> 
> 
> *   system()函数
> 
> system在执行系统外部命令时，直接将结果输出到浏览器，正确返回true，否则为false
> 
> 
> *   passthru()函数
> 
> passthru直接将结果输出到浏览器，不返回任何值
> 
> 
> *   shell_exec()函数
> 
> string shell_exec(string $cmd) 直接执行命令$cmd
3.扫描端口-查看可利用端口

常见的可利用端口有4899-Radmin，1433-SQL Server，3306-Mysql，43958-Serv-U，3389-远程连接终端

4899 1433 3306 43958都需要知道各自账号密码才能进一步进行提权，1433 3306都可以在网站的配置文件中找到账号密码，43958的账号密码一般都是默认的
> Serv-U默认账号密码
> 
> 
> 用户名： LocalAdministrator
> 
> 
> 口令：#l@$ak#.lk;0@P
之后我还会具体详细学习，如何利用脚本和服务器上的其他服务进行提权。