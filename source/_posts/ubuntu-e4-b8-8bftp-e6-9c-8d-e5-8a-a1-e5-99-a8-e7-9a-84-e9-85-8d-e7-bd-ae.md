---
title: Ubuntu下FTP服务器的配置
tags:
  - Web安全
  - 服务器配置
id: 68
categories:
  - Web安全
  - 服务器
date: 2016-07-28 10:20:21
---

# **Ubuntu下FTP服务器的配置**

![image](http://img.wdjimg.com/mms/icon/v1/0/8e/62c1cec239733d94ade53e8c37dae8e0_256_256.png)

## 导语

> 有了服务器以后，就想往上面上传点东西了，于是乎就有了接下来的故事 `FTP服务器搭建之旅`
> 
> 当然啦首先你得现有一个服务器~~

<!--more-->
### 正文

#### 1\. 安装vsftpd

打开**终端**输入`sudo apt-get install vsftpd`安装vsftpd服务

#### 2\. 判断vsftpd是否安装成功

在**终端**中输入`sudo service vsftpd restart`重启vsftpd服务

#### 3\. 新建“/home/uftp”目录作为主目录

打开**终端** 输入`sudo mkdir /home/uftp` 成功创建目录

#### 4\. 新建用户uftp并设置密码

在**终端**中输入`sudo useradd -d /home/uftp -s /bin/bash uftp`
输入`sudo passwd uftp`设置用户uftp的密码

#### 5.修改配置文件/etc/vsftpd.conf

修改配置文件**vsftpd.conf**

    在配置文件中加入如下配置
    <span class="hljs-constant">userlist_deny</span>=NO
    <span class="hljs-constant">userlist_enable</span>=YES
    <span class="hljs-constant">userlist_file</span>=/etc/allowed_users //文件中的用户及可以连接服务器的用户

#### 6.没有allowed_users的需手动创建

#### 7.查看文件/etc/ftpusers

查看文件_ftpusers_看如果存在用户名uftp,需要删除此用户，此文件中的用户即禁止连接服务器的用户

* * *

通过以上的操作即可完成基本的**FTP服务器的配置**,需要注意的是在配置的过程中如果需要**上传文件**，则需要在配置文件**vsftpd.conf**中添加上对应的写权限`write_enable=YES`才能使其正常的进行连接上传 ![image](http://123.206.79.232/img/FTP1.png)

* * *

此次blog,是我第一次写blog，同样也是第一次使用**markdow**，感觉确实很赞，简单好用，以后我的博客还会不断更新**Web**和**Web安全**方面的内容