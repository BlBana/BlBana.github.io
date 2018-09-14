---
title: 论如何修改Win10密码
categories: 
  - Win10
tags:
  - 运维
  - 修改密码
toc: true 文章目录
author: BlBana
date: 2016-12-14 15:36:33
description:
comments:
original:
permalink:
---
## 一次坑爹的密码被改经历
> 如今我也是有故事的人了。前天晚上我们在实验室搞到一点多用完电脑直接关机，和友友们一起出去撸了个串，回宿舍就睡了，等我第二天用电脑的时候，开机输入密码，提示错误，还以为是手滑，我一个一个重新的输入，卧槽！！！还是不对，试了十几遍...MD密码被改了！！！坑爹了...电脑上重要的东西不能重装系统，赶紧查资料把密码改了回来...（醉了欸，成功登上电脑后还分析了一波日志，进程之类的，看看是什么情况...）
 
<!-- more -->

**找回Winodws密码有两种方法，都是需要U盘进行启动的**

### 0x01 利用PE盘重置密码

> 第一种方法就是利用制作的PE启动盘进行修改，这种方法比较简单，从网上找现成的教程，制作一个PE的启动盘，然后就能开搞了。

首先可以看到，我的虚拟机里的XP系统是有密码的，这时候不知道密码的我们是上不去的。
![Alt text](http://blog.blbana.cc/img/hexo/Win10/1.png)

**现在我们开始来重置密码**

先将我们的启动项改为U盘启动，将系统从我们的U盘中打开。
![Alt text](http://blog.blbana.cc/img/hexo/Win10/2.png)

直接打开PE盘的第一个选项，**Pe Win8PEx86精简版**（贼难打的几个字...）
![Alt text](http://blog.blbana.cc/img/hexo/Win10/3.png)

进入PE系统中就能看见有一个重置登录密码的软件
![Alt text](http://blog.blbana.cc/img/hexo/Win10/4.png)

直接打开软件，选择我们药重置的用户，勾上重置密码的选项
![Alt text](http://blog.blbana.cc/img/hexo/Win10/5.png)

保存，我们的密码就重置成功了，再次重启系统就可以直接进入系统，没有了密码验证的过程了...**感觉电脑好危险**
![Alt text](http://blog.blbana.cc/img/hexo/Win10/6.png)
![Alt text](http://blog.blbana.cc/img/hexo/Win10/7.png)
OK啦

### 0x02 利用Windows启动盘
> 第二种更改方法相对的麻烦了一些，没有了第一步那么现成的工具，一键就可以重置密码，第二种方法主要的思想就是，利用U盘中的系统，去替换我们系统盘中的连按shift的粘贴功能为启动我们具有管理员权限的命令行。

刚开始和第一种方式一样，进入U盘启动当中，我们会进到安装系统的界面当中，按shift+F10，启动一个CMD的命令行当中。
![Alt text](http://blog.blbana.cc/img/hexo/Win10/8.png)

进入命令行后，选择自己本机C盘所在的盘符（**这时候C盘可能已经变成其他盘符**）我这里是E盘。
![Alt text](http://blog.blbana.cc/img/hexo/Win10/9.png)

进入到以上的目录当中，将系统中用于启动粘滞键sethc.exe重命名为sethc_bak.exe，再将cmd.exe替换成sethc.exe。
`ren sethc.exe sethc_bak.exe`
`ren cmd.exe sethc.exe`
替换完毕以后，重新启动系统。

到了登陆界面的时候，我们连续按shift键后，发现cmd命令行打开，并且是管理员的身份。
![Alt text](http://blog.blbana.cc/img/hexo/Win10/10.png)

我们直接使用命令修改当前用户的登陆密码
`net user BlBana password`

修改完成后，我们可以用新密码进行登陆。

> 整个过程就是这个样子的，我是用的第二种方法进行的密码修改，这里我发现，在我使用U盘来替换sethc.exe文件的时候很顺利，但是到了进入系统以后想要把文件替换回来的时候，提示我的权限不够了（**已经切换到了管理员**）。
> 这里我的猜测是，当我们利用U盘启动系统的时候，我们本机的系统里的文件对于U盘系统来说，只是一些普通的文件而已，缺少了系统的保护，我们可以随意的进行更改。

---
以上两种方法都是通过物理介质来对计算机密码进行了修改，但是实际的过程中，我们很难实际接触到物理机，所以以上的方法当我们忘记自己开机密码的时候使用挺不错的选择，避免了重装系统导致文件的丢失。