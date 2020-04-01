---
title: Fastjson反序列化漏洞基础
categories:
  - Web安全
tags:
  - Fastjson反序列化
  - 漏洞分析
  - JavaSec
toc: true 文章目录
author: BlBana
date: 2020-03-29 20:44:27
description:
comments:
original:
permalink:

---

> 本篇是Java反序列化漏洞开坑的第二篇，主要是介绍了下Fastjson主要API的基本使用方式，序列化/反序列化的一些特性，以及**Map和User**类型在反序列化时的流程分析，对一些核心的反序列化过程进行了跟进，主要还是为了熟悉不同类型的Json字符串在反序列化过程中构造方法，setter方法，getter方法调用情况，帮助理解PoC的构造思路。在此基础上，下一篇接着会对`Templateslmpl类` 的利用链进行调试分析。

<!-- more -->

------

# Fastjson反序列化漏洞基础

# Fastjson简介

`Fastjson`是阿里巴巴的开源JSON解析库，它可以解析JSON格式的字符串，支持将Java Bean序列化为JSON字符串，也可以从JSON字符串反序列化到JavaBean。`Fastjson`在阿里巴巴大规模使用，在数万台服务器上部署，Fastjson在业界被广泛接受。在2012年被开源中国评选为最受欢迎的国产开源软件之一。出现安全问题影响范围很广。

# 序列化/反序列化基础知识

```xml
# 使用1.2.23版本进行测试
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>fastjson</artifactId>
  <version>1.2.23</version>
</dependency>
```

## API介绍

- 序列化：`String text = JSON.toJSONString(obj);`
- 反序列化：
  - `JSON.parseObject` 返回JSONObject类型
  - `JSON.parse` 返回实际类型对象

## 使用方法

- User类

  ```java
  public class User {
      private String name;
      private int age;
  
      public User() {
          System.out.println("Called in 构造方法");
      }
  
      public String getName() {
          System.out.println("Called in getName()");
          return name;
      }
  
      public void setName(String name) {
          System.out.println("Called in setName()");
          this.name = name;
      }
  
      public int getAge() {
          System.out.println("Called in getAge()");
          return age;
      }
  
      public void setAge(int age) {
          System.out.println("Called in setAge()");
          this.age = age;
      }
  
      @Override
      public String toString() {
          return "User{" +
                  "name='" + name + '\'' +
                  ", age=" + age +
                  '}';
      }
  }
  ```

- 序列化

  ```java
  import com.alibaba.fastjson.JSON;
  import com.alibaba.fastjson.serializer.SerializerFeature;
  
  public class FJSer {
      public static void main(String[] args) {
          User user = new User();
          user.setAge(18);
          user.setName("BlBana");
          String jsonString = JSON.toJSONString(user, SerializerFeature.WriteClassName);
          System.out.println(jsonString);
      }
  }
  
  // 返回结果
  // Called in 构造方法
  // Called in setAge()
  // Called in setName()
  // Called in getAge()
  // Called in getName()
  // {"@type":"User","age":18,"name":"BlBana"}
  ```

- 反序列化

  ```java
  import com.alibaba.fastjson.JSON;
  
  public class FJTest {
      public static void main(String[] args) {
          String userString = "{\"@type\":\"User\" ,\"age\":25,\"name\":\"blbana\"}";
          User user = JSON.parseObject(userString, User.class);
          System.out.println(user);
          System.out.println(user.getClass().getName());
      }
  }
  
  // Called in 构造方法
  // Called in setAge()
  // Called in setName()
  // User{name='blbana', age=25}
  // User
  ```

  `SerializerFeature.WriteClassName` 用于在序列化后的字符串中添加@type属性，存放对象类型。在反序列化时，可以根据@type定义的类型解析生成对象。

## Fastjson特性

- User类

  ```java
  public class User {
      private int age;
      private String name;
      private String sex;
      public String address;
  
      public User() {
          System.out.println("Called in User()");
      }
  
      public int getAge() {
          System.out.println("Called in getAge()");
          return age;
      }
  
      public String getName() {
          System.out.println("Called in getName()");
          return name;
      }
  
      public String getSex() {
          return sex;
      }
  
      public void setSex(String sex) {
          this.sex = sex;
      }
  
      @Override
      public String toString() {
          return "User{" +
                  "age=" + age +
                  ", name='" + name + '\'' +
                  ", sex='" + sex + '\'' +
                  ", address='" + address + '\'' +
                  '}';
      }
  }
  ```

- 反序列化

  ```java
  import com.alibaba.fastjson.JSON;
  
  public class FJTest {
      public static void main(String[] args) {
          String userString = "{\"@type\":\"User\" ,\"age\":25,\"name\":\"blbana\",\"address\": \"xian\", \"sex\": \"man\"}";
          User user = JSON.parseObject(userString, User.class);
          System.out.println(user);
          System.out.println(user.getClass().getName());
      }
  }
  
  // Called in User()
  // Called in setSex()
  // User{age=0, name='null', sex='man', address='xian'}
  // User
  ```

  四个属性，公有属性有无getter/setter都可以反序列化成功，私有属性有getter/setter反序列化成功，私有属性无setter反序列化失败：

  - `private name，getter，未反序列化`
  - `private age，getter，未反序列化`
  - `private sex，getter/setter，反序列化成功`
  - `public address，反序列化成功`

### Feature.SupportNonPublicField

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

public class FJTest {
    public static void main(String[] args) {
        String userString = "{\"@type\":\"User\" ,\"age\":25,\"name\":\"blbana\",\"address\": \"xian\", \"sex\": \"man\"}";
        User user = JSON.parseObject(userString, User.class, Feature.SupportNonPublicField);
        System.out.println(user);
        System.out.println(user.getClass().getName());
    }
}

// Called in User()
// Called in setSex()
// User{age=25, name='blbana', sex='man', address='xian'}
// User
```

上述例子中，不含有setter的私有属性无法反序列化，给parseObject方法加入`Feature.SupportNonPublicField`属性后，即可完成age和name两个私有属性的反序列化。

### 反序列化不同属性对比

主要是在`parse/parseObject`进行反序列化时，通过传入不同的属性对比反序列化的过程和反序列化结果有什么不同，找到类setter方法，getter方法，构造方法的调用条件，用于后期构造PoC，这里主要有三个属性影响：

- 有无Class类型
- 不同Class类型
- 有无Feature.SupportNonPublicField属性

------

```java
import java.util.Properties;

public class User {
    private int age;
    private String name;
    private String sex;
    private Properties properties;
    public String address;

    public User() {
        System.out.println("Called in User()");
    }

    public int getAge() {
        System.out.println("Called in getAge()");
        return age;
    }

    public String getName() {
        System.out.println("Called in getName()");
        return name;
    }

    public String getSex() {
        System.out.println("Called in getSex()");
        return sex;
    }

    public void setSex(String sex) {
        System.out.println("Called in setSex()");
        this.sex = sex;
    }

    public Properties getProperties() {
        System.out.println("Called in getProperties()");
        return properties;
    }

    @Override
    public String toString() {
        return "User{" +
                "age=" + age +
                ", name='" + name + '\'' +
                ", sex='" + sex + '\'' +
                ", properties=" + properties +
                ", address='" + address + '\'' +
                '}';
    }
}
```

1. **有无Class类型/不同Class类型**

- 无Class类型

  ```java
  import com.alibaba.fastjson.JSON;
  
  public class FJTest {
      public static void main(String[] args) {
          String userString = "{\"@type\": \"User\" ,\"age\": 18,\"name\": \"blbana\",\"address\": \"xian\", \"sex\": \"man\", \"properties\":{}}";
          Object user = JSON.parseObject(userString);
  // JSONObject user = JSON.parseObject(userString);
          System.out.println(user);
          System.out.println(user.getClass().getName());
      }
  }
  //Called in User()
  //Called in setSex()
  //Called in getProperties()
  //Called in getAge()
  //Called in getName()
  //Called in getProperties()
  //Called in getSex()
  //{"address":"xian","sex":"man","age":0}
  //com.alibaba.fastjson.JSONObject
  ```

  这种情况下调用了构造方法，所有属性的getter方法，所有属性的setter方法，其中getProperties被调用两次；定义类型无论是Object，还是JSONObject结果都一样，返回的类型都为JSONObject。

- 有Class类型

  ```java
  import com.alibaba.fastjson.JSON;
  import com.alibaba.fastjson.JSONObject;
  
  public class FJTest {
      public static void main(String[] args) {
          String userString = "{\"@type\": \"User\" ,\"age\": 18,\"name\": \"blbana\",\"address\": \"xian\", \"sex\": \"man\", \"properties\":{}}";
  //        Object object = JSON.parseObject(userString, Object.class);
  //        Object object = JSON.parseObject(userString, User.class);
          User object = JSON.parseObject(userString, User.class);
          System.out.println(object);
          System.out.println(object.getClass().getName());
      }
  }
  //Called in User()
  ////Called in setSex()
  ////Called in getProperties()
  ////User{age=0, name='null', sex='man', properties=null, address='xian'}
  ////User
  ```

  在有Class类型时，调用构造方法，所有属性的setter方法，properties属性的getter方法；定义类型无论是Object，User返回类型都是User类型。反序列化时根据@type制定类型进行解析，定义了对象的类型。

------

1. 有无Feature.SupportNonPublicField属性

   ```java
   import com.alibaba.fastjson.JSON;
   import com.alibaba.fastjson.JSONObject;
   import com.alibaba.fastjson.parser.Feature;
   
   public class FJTest {
       public static void main(String[] args) {
           String userString = "{\"@type\": \"User\" ,\"age\": 18,\"name\": \"blbana\",\"address\": \"xian\", \"sex\": \"man\", \"properties\":{}}";
           Object object = JSON.parseObject(userString, Object.class, Feature.SupportNonPublicField);
   //        Object object = JSON.parseObject(userString, User.class, Feature.SupportNonPublicField);
   //        User object = JSON.parseObject(userString, User.class, Feature.SupportNonPublicField);
           System.out.println(object);
           System.out.println(object.getClass().getName());
       }
   }
   //Called in User()
   //Called in setSex()
   //Called in getProperties()
   //User{age=18, name='blbana', sex='man', properties=null, address='xian'}
   //User
   ```

   加入`Feature.SupportNonPublicField` 属性后，调用构造方法，所有属性的setter方法，properties属性的getter方法；返回类型都为User类型，并且不含有setter的私有属性也反序列化成功。

------

这里直接引用一下**[mi1k7ea的总结](https://www.mi1k7ea.com/2019/11/03/Fastjson%E7%B3%BB%E5%88%97%E4%B8%80%E2%80%94%E2%80%94%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/#%E5%B0%8F%E7%BB%93)**，写的非常详细：

- [https://www.mi1k7ea.com/2019/11/03/Fastjson系列一——反序列化漏洞基本原理/#小结](https://www.mi1k7ea.com/2019/11/03/Fastjson%E7%B3%BB%E5%88%97%E4%B8%80%E2%80%94%E2%80%94%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/#%E5%B0%8F%E7%BB%93)

### Parse 和 ParseObject区别

两者的主要区别就是，**parseObject**返回的是**JSONObject**类型，而**parse**返回的是**实际类型的对象**。

这里是调用parseObject返回值的情况，实际parseObject调用的也是parse方法，只不过在返回之前，将目标对象转换为了JSONObect类型。

```java
// parseObject返回前会将对象转换为JSONObject类型
return obj instanceof JSONObject ? (JSONObject)obj : (JSONObject)toJSON(obj);
```

# 反序列化流程分析

## Map反序列化

```java
public class FastjsonTest {
    /**
     * FastJson Test Case
     * @param args
     */
    public static void main(String[] args) {
        Map<String, Object> map = new HashMap<>();
        map.put("key1", "One");
        map.put("key2", "Two");

        // 序列化
        String mapJson = JSON.toJSONString(map);

				// 反序列化
        Object a2 = JSON.parseObject(mapJson);
    }
}
```

### 关键类

跟进`parseObject`方法，其中在`parser`中有几个比较关键的类

- `JSONLexe`—— 处理Json分词，next()可以获取Json字符串的下一个字符
- `ParserConfig`—— 包含解析配置，反序列化器，标签等各类配置信息
- `JavaBeanDeserializer` —— JavaBean反序列化类
- `JSONScanner` —— 负责扫描和获取json字符串中的Token并返回
- `ObjectDeserializer` —— 负责将json字符串反序列化，与JavaBean有关系，内置各种类型的反序列化器

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/1.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/1.png)

在只给入`Json String`的情况下，默认按照`Map`类型进行解析

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/2.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/2.png)

解析过程中使用`skipWhitespace`方法空白字符一律跳过处理

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/3.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/3.png)

继续跟进到`scanSymbol`方法中，循环获取`lexer`分词器中的字符并根据字符的ASCII码值获取到一个`hash`整数，根据hash从预先处理好`table`中的将双引号中的`key`值获取并返回，此时获取到第一个键名`"key1"`

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/4.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/4.png)

获取到"key1"键名后，使用`stringVal`方法获取到键值，并将键值对放入到`JSONObect object`的`map`中，至此已经将`Json`字符串中第一个键值对反序列化并放入`JSONObject`对象中，再判断到`Json`字符串未解析完时，循环开始下一个键值对的解析

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/5.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/5.png)

## User反序列化

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

public class FJTest {
    public static void main(String[] args) {
        String userString = "{\"@type\":\"User\",\"sex\": \"man\", \"age\":25,\"name\":\"blbana\",\"address\": \"xian\", \"properties\": {}}";
        User object = JSON.parseObject(userString, User.class, Feature.SupportNonPublicField);
        System.out.println(object);
        System.out.println(object.getClass().getName());
    }
}
```

跟进`parseObject`方法，进入到`DefaultJSONParser.parseObject` 方法，根据`Class`类型获取对象的反序列化器`derializer`，并进行反序列化操作

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/6.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/6.png)

跟进`this.config.getDeserializer(type)`获取反序列化器的流程，先尝试从HashMap中获取User类型的预设反序列化器，由于未找到，进入`this.getDeserializer()`方法

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/7.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/7.png)

继续跟进，`this.getDeserializer()`中主要通过比对type类型寻找合适的反序列化器，其中有一次`this.denyList`黑名单判断，黑名单中的类出现会抛出异常

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/8.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/8.png)

对type Class类型，ClassName进行一系列比对，由于未能在预设的反序列化器中找到合适的，根据Class类型调用`this.createJavaBeanDeserializer` 创建`JavaBeanDeserializer`

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/9.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/9.png)

其中`beanInfo`中存放着**User类的Class对象，User类构造器，一些字段的Class对象及其构造器**，经过一些判断后，暂且未用到这些信息，不过这种存放bean信息的类倒是挺有意思的，之后会跟进下`build`方法具体内容

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/10.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/10.png)

进入到`JavaBeanDeserializer` 类中

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/11.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/11.png)

再次生成`beanInfo`对象，并循环根据`fieldInfo`信息给每个字段生成对应类型的`fieldDeserializer`并存放到`this.sortedFieldDeserializers`和`this.fieldDeserializers`

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/12.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/12.png)

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/13.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/13.png)

`JavaBeanDeserializer`生成完毕，回到主流程开始调用`derializer.deserialze`方法

> `this.extraFieldDeserializers`中存放之前`this.sortedFieldDeserializers`中未存放的两个属性，后面会跟进下属性为什么会分到两个不同的`Map`中

- `this.extraFieldDeserializers`
  - name
  - age
- `this.sortedFieldDeserializers`
  - address
  - sex
  - properties

先会实例化`User`对象，紧接着会对User的字段进行反序列化并赋值到User对象的对应属性中

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/14.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/14.png)

循环将`this.extraFieldDeserializers`和`this.sortedFieldDeserializers`中准备好的`Field`使用`parseField`方法解析到**User object**中去，进一步跟进`parseField`方法

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/15.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/15.png)

按照key的类型，利用**“智能匹配”**功能，找到属性的反序列化器，调用反序列化器的`parseField`方法

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/16.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/16.png)

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/17.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/17.png)

`parseField`方法中，对属性进行反序列化处理，调用`this.fieldValueDeserilizer.deserialze` 方法

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/18.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/18.png)

获取到value指后调用`this.setValue`方法，将属性值加入到object中

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/19.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/19.png)

- `setValue`对于能获取到属性Method对象的，直接利用反射将值**set**到属性中

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/20.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/20.png)

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/21.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/21.png)

- `setValue`对于无法获取的Method对象的，使用**field.set**给属性设置一个新值

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/22.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/22.png)

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/23.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/23.png)

也就是说在属性反序列化获取到值后，都会调用setValue将值赋值到object中对应的属性中：

- 对于有setter的，利用反射调用**setter**方法赋值；
- 对于没有setter的，利用字段**field.set**的方式给属性赋值；

当所有Field对象都实例化并添加到User对象object中后，返回object，结束反序列化流程。

## Note：关于JavaBeanInfo的build方法

之前可以看到每次反序列化时，`getProperties`方法都会在这里被调用，这里跟一下Method是如何被赋值到`this.fieldInfo.method`的，主要发生在build方法中

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/24.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/24.png)

在build方法中，首先就会利用反射获取到类的`字段和方法`列表，以及`User`的构造方法

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/25.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/FastjsonBasic/25.png)

```java
// set 方法
for(i = 0; i < var29; ++i) {
	method = var30[i];
	String methodName = method.getName();
	if (methodName.length() >= 4 && !Modifier.isStatic(method.getModifiers()) && (method.getReturnType().equals(Void.TYPE) || method.getReturnType().equals(method.getDeclaringClass()))) {
	   Class<?>[] types = method.getParameterTypes();
     if (types.length == 1) {

// get方法
if (methodName.length() >= 4 && !Modifier.isStatic(method.getModifiers()) && methodName.startsWith("get") && Character.isUpperCase(methodName.charAt(3)) && method.getParameterTypes().length == 0 && (Collection.class.isAssignableFrom(method.getReturnType()) || Map.class.isAssignableFrom(method.getReturnType()) || AtomicBoolean.class == method.getReturnType() || AtomicInteger.class == method.getReturnType() || AtomicLong.class == method.getReturnType())) {
```

之后`method`被循环取出，开始判断：

### set方法

1. 方法名要大于等于4；
2. 非静态方法；
3. 返回值未void或者返回当前类；
4. 参数只有一个

### get方法

1. 方法名要大于等于4；
2. 非静态方法；
3. 以get开头并且第四个字母为大写；
4. 参数个数为0；
5. Collection & Map & AtomicBoolean & AtomicInteger & AtomicLong 方法返回类型

由于`getProperties`符合条件被放入到**List**中，在后续反序列化字段的时候这个**method**会被调用。

# 问题记录

1. 存放`key`的`symbolTable`是何时解析的？

   在SymbolTable对象生成时，**$ref** 和 **@type** 两个关键字已经被预设进去。

2. 存放`value`的方法是如何获取到值得？

   通过循环获取到目标值得开始和结束索引位置，通过subString方法截取目标值

# 参考链接

- <https://www.mi1k7ea.com/2019/11/03/Fastjson%E7%B3%BB%E5%88%97%E4%B8%80%E2%80%94%E2%80%94%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/>
- <https://www.mi1k7ea.com/2019/11/07/Fastjson%E7%B3%BB%E5%88%97%E4%BA%8C%E2%80%94%E2%80%941-2-22-1-2-24%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/>
- <https://p0sec.net/index.php/archives/123/>