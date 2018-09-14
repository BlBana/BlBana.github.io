---
title: L-CTF writeup学到的新姿势
id: 258
categories:
  - 未分类
tags:
---

## 0x01 导语

L-ctf已经结束了，看完了writeup发现有许多不会的地方，回头一个一个的补，学会了很多新的CTF解题思路和脑洞，现在先总结一下，码下来，可以多看看

## 0x02 正文

a.Web 200 睡过了

关键的地方就是学会了刚爆出来的一个PHP反序列化漏洞的利用方法，PHP爆出来了一个漏洞，当序列化字符串中表示对象的个数的值大于真实的属性个数会跳过__wakeup的执行。

<span style="color: #ff0000;">此处补代码</span>

![](http://www.blbana.cc/wp-content/uploads/2016/10/8104f58caa33a27670098f239338ac6b.png)

正常情况应该是都被置空，但是当我们将对象个数改大以后就可以跳过置空这个环节

![](http://www.blbana.cc/wp-content/uploads/2016/10/aae6c9f8f6f53aba571bdf355338828d.png)

从而造成了对象注入，在SugarCRM中可导致直接写入一句话getshell

`b.绕过open_basedir的限制 `

尝试了一下，使用了open_basedir后，即使连上caidao也无法访问其他目录下的文件

顺便试了一下caidao无法上传文件的问题，我电脑里这个目录有四个用户~

![](http://www.blbana.cc/wp-content/uploads/2016/10/e21d3df62006f4adfd104384296dd927.png)

<span style="color: #00ff00;">我尝试分别禁用了1，3，4这三个用户对此目录的写入权限，都会导致caidao上传文件的失败，但是只禁用system的写入权限时不影响，说明我从web访问服务器的时候，是以1，3，4这三个用户的权限进行访问的，禁用任意一个的写入权限，都会导致无法上传（我用的是wamp运行的程序）</span>

https://www.leavesongs.com/PHP/php-bypass-open-basedir-list-directory.html大牛博客有详细的绕过方法。

b.web 250 苏打学姐的网站