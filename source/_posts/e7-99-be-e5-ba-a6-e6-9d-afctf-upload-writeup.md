---
title: 百度杯CTF--第一周 writeup
tags:
  - PHP
  - 上传漏洞
  - 上传绕过方法
  - 绕过过滤
id: 229
categories:
  - PHP
  - 上传漏洞
date: 2016-09-05 17:53:33
---

## 目的

![](http://www.blbana.cc/wp-content/uploads/2016/09/7cc41055b2df1a9b480183913b32809e.png)

这道题开始给的提示是flag.php中有flag，因此我们的目的就是读这个flag.php。
<!--more-->
## 解题思路

![](http://www.blbana.cc/wp-content/uploads/2016/09/d62522878aeaa5918594e519fac98dbe.png)

进来后发现是一个上传界面，我们先尝试上传一些正常文件

![](http://www.blbana.cc/wp-content/uploads/2016/09/e18719d581bf9f4d9f59413dcc0b6b00.png)

![](http://www.blbana.cc/wp-content/uploads/2016/09/1264b5c02738f094a5af03db9d3a4fda.png)

是可以正常上传的，现在我们尝试上传一个.php的文件

![](http://www.blbana.cc/wp-content/uploads/2016/09/d4b844ae06eb0072237b29a27a769747.png)

上传后，发现我们代码里的&lt;?php被过滤了，导致代码无法正常解析，被当作字符串输出了。

经过测试发现被过滤的是&lt;?和php，并且无法动态拼接绕过，但是php只是过滤了小写，大写字母并没有做出任何处理，我们可以通过大小写绕过php的过滤，&lt;?无论怎么拼接，都会被循环过滤掉，因此需要换一种思路。

经过查询发现，php的标准标签其实有两种：

1.&lt;?php                                            ?&gt;                    这里&lt;?会被过滤行不通；

2.&lt;script language=php&gt; &lt;/script&gt;            这里不存在&lt;?，我们不怕php无法被解析了

* * *

现在我们需要构造一个没有php和&lt;?的脚本去上传

![](http://www.blbana.cc/wp-content/uploads/2016/09/c244d520c389e470226ab10cee68f025.png)

![](http://www.blbana.cc/wp-content/uploads/2016/09/fcb6e581b440477aab8261b34a7e0f8f.png)

OK，我们代码正常运行了。现在可以去尝试读取flag.php的文件了。

![](http://www.blbana.cc/wp-content/uploads/2016/09/4ec5288d2dd719bc67f7d4f629118883.png)

由于提示上说，flag在flag.php中，php需要为小写，否则无法读到正确的文件，但是我们代码里要是存在php的话就会被过滤掉，因此利用url编码，绕过开始对php的过滤。

成功读到文件

![](http://www.blbana.cc/wp-content/uploads/2016/09/ddfaea289d90dd31b0c6e57edb28963c.png)

* * *

当我们上传一句话的时候，就会被拦截，这时可以利用burp抓包进行截断绕过上传，成功上传含有一句话的脚本，尝试使用caidao连接，报错405 not allow，看来无法使用caidao连接。

这道题的关键是要知道php的另外一种标准标签的形式，否则无法绕过开始对&lt;?的绕过。

## YeserCMS

![](http://www.blbana.cc/wp-content/uploads/2016/09/f1ba0114e441d777824bc0c17c368f35.png)

经搜索这个Yesercms并不存在，因此可能是某种cms的二次开发得到的，发现是cmseasy，于是从网上找到了许多关于这个cms的漏洞。

最后在/celive/live/header.php目录下存在注入，直接post提交payload

`xajax=Postdata&amp;xajaxargs[0]=<q>detail=xxxxxx%2527%252C%2528UpdateXML%25281%252CCONCAT%25280x5b%252Cmid%2528%2528SELECT%252f%252a%252a%252fGROUP_CONCAT%2528concat%2528username%252C%2527%257C%2527%252Cpassword%2529%2529%2520from%2520yesercms_user%2529%252C1%252C32%2529%252C0x5d%2529%252C1%2529%2529%252CNULL%252CNULL%252CNULL%252CNULL%252CNULL%252CNULL%2529--%2520</q>
`

<span style="color: #ff0000;">获取密码的过程中，由于显示长度的限制，最多只能显示25位的md5值，这时候可以改变一下读取的起始位置，保证之后的内容能够正常读取获取到32位</span>

获得账号为admin    密码为Yeser231

![](http://www.blbana.cc/wp-content/uploads/2016/09/2b9986df2e490c3bebbc0958cb617e32.png)

进入后台

在编辑模板处存在文件包含漏洞

![](http://www.blbana.cc/wp-content/uploads/2016/09/ca2b6ff33d00f9263080313030315390.png)

![](http://www.blbana.cc/wp-content/uploads/2016/09/859d088a4999c107559155b90ace5ad1.png)

抓包修改参数id为flag.php的路径../../flag.php成功获取flag的值

![](http://www.blbana.cc/wp-content/uploads/2016/09/539fb29e4179e2a8e02719f332836608.png)