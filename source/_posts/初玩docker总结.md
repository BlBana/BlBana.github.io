---
title: 初玩docker总结
categories:
  - docker
tags:
  - CTF
  - docker
  - 服务器配置
  - 沙盒
toc: true 文章目录
author: BlBana
date: 2016-10-08 13:14:06
description:
comments:
original:
permalink:
---

## 导语：

这两天因为要搭建一个ctf的平台，需要给web题做一个沙盒，就想到了用docker来配置，防止服务器被拿下，第一次玩docker总结了一些注意事项。
<!-- more -->
## 正文：

1.docker进入运行中的容器：
```bash
sudo docker inspect -f {{.State.Pid}} 44fc0f0582d9
sudo nsenter --target 3326 --mount --uts --ipc --net --pid
```
这样子可以成功进入容器当中，更改玩Web服务器的相关配置以后，需要使用命令sudo docker restart ID , 重启容器才可以生效，但是如果使用stop命令暂停容器，在使用start命令重启的话，会导致配置还原。

要想保存更改过后的容器为镜像的话可以使用commit命令来保存：

![](http://www.blbana.cc/wp-content/uploads/2016/10/48416f278a0da16dd7e1d5c997d9800e.png)

记录好更改后的容器ID，使用命令：sudo docker commit &lt;ID&gt; &lt;name&gt;

就OK了

2.挂在本地目录到容器中

sudo docker run -d -v /host/path:/path_in_container -p host/port:container/port container&lt;name&gt;  就可以把本机目录挂在到容器中，并运行在指定的端口上

总结的暂时就这么多，之后遇到问题还会继续放上~