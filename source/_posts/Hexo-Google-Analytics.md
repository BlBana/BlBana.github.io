---
title: Hexo Google Analytics
categories:
  - Hexo
tags:
  - Hexo
  - Linux
  - SEO
toc: true 文章目录
author: BlBana
date: 2020-03-30 19:53:14
description:
comments:
original:
permalink:
---

> 为了这小po站，做了一点小小的优化，加入了 Google Analytics 和 Google Search，这里做点记录 ... 终于把HTTPS加上了，我怕是个假安全 ...

<!-- more -->

---

# Google Analytics

> 主要为了能定期看看博客的访问情况

先到[Google Analytics首页](https://www.google.com/analytics)建立账户，添加一个新的媒体资源，完成一系列信息填写后，可以获取到追踪代码，只需要将其放到网站全局代码头中即可。

比如下面这个就是我的站点追踪代码，可以看到Google使用id来表示媒体资源`UA-XXXX-X` 

```javascript
<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-162189802-1"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-162189802-1');
</script>
```

我这里用的hexo主题是archer，在`themes/archar/_config.yml`文件中已经预留了id的配置项：

```yaml
# Google analytics
google_analytics: UA-162189802-1
```

这个配置最终会引入到`themes/archer/layout/_partial/base-head.ejs` 中，发现之前预设代码请求地址发生了变化，修改替换即可：

- 预设代码

  ```javascript
  <script>
  	  (function (i, s, o, g, r, a, m) {
  	  i['GoogleAnalyticsObject'] = r; i[r] = i[r] || function () {
  	  (i[r].q = i[r].q || []).push(arguments)
  	  }, i[r].l = 1 * new Date(); a = s.createElement(o),
  	  m = s.getElementsByTagName(o)[0]; a.async = 1; a.src = g; m.parentNode.insertBefore(a, m)
  	  })(window, document, 'script', 'https://www.google-analytics.com/analytics.js', 'ga');
  	  ga('create', '<%- theme.google_analytics %>', 'auto');
  	  ga('send', 'pageview');
  </script>
  ```

- 替换代码

  ```javascript
  <!-- Global site tag (gtag.js) - Google Analytics -->
  <script async src="https://www.googletagmanager.com/gtag/js?id=<%= theme.google_analytics %>"></script>
  <script>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', '<%= theme.google_analytics %>');
  </script>
  ```

替换后编译上传即可，差不多几分钟就能在实时页面上看到效果：

- 概览
- 实时数据
- 访问会话
- 会话时长
- 流量渠道/来源媒介
- 地理位置
- 页面浏览量
- 访问设备
- ...

> 辣鸡Baidu再见 ！

![Hexo%20Google%20Analytics/Untitled.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/0.png)

# Google Search Console

> 主要优化下搜索

`Google Search Console`地址注册账号，并将自己的域名或者子域名进行所有权验证

- 域名需要使用DNS解析记录验证，添加TXT记录验证即可
- 子域名支持不同方式进行验证，具体可以自行查看

验证完成后，大约需要一天的时间处理数据，这里主要还是看Google是否已经收录了你的域名，比如我的`blbana.cc`域名上没有什么实际内容，并未被Google收录；但我`drops.blbana.cc`因为解析到了博客上面，验证完成后已经可以看到之前的一些搜索记录信息了

![Hexo%20Google%20Analytics/Untitled%201.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/1.png)

![Hexo%20Google%20Analytics/Untitled%202.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/2.png)

为了方便Google能及时收录网站的新增页面，可以使用hexo的插件`hexo-generator-sitemap` 帮助生成`sitemap.xml`文件并提交到[Google站点地图](https://search.google.com/search-console/sitemaps)

![Hexo%20Google%20Analytics/Untitled%203.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/3.png)

```bash
# 安装命令
npm install hexo-generator-sitemap --save
```

# Github Pages HTTPS

> 这里把博客的HTTPS也加上了

## Github自带HTTPS配置

配置十分简单，直接在`Setting→Github Pages`中开启HTTPS即可

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/4.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/4.png)

等上几分钟，进入博客HTTPS生效，可以看到Github Pages自动帮我们的域名在[Let's Encrypt](https://letsencrypt.org/zh-cn/getting-started/)上申请了免费证书

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/5.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/5.png)

![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/6.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/6.png)

## Nginx反向代理启用HTTPS

除了这种方式，如果自己想折腾的话也可以去[Let's Encrypt](https://letsencrypt.org/zh-cn/getting-started/)上申请一个证书，然后自己**搭个Nginx配上HTTPS在做个反向代理**即可。

这里有一篇参考文章：[反向代理Github Pages启用HTTPS](https://imzlp.me/posts/18841/)，自己做代理需要注意的就是证书续签的问题了，Let's Encrypt证书有效期默认是90天，搞个Crontab每月1号定时自动续签下就OK了。

```bash
0 0 1 * * /root/certbot/certbot-auto renew --pre-hook "service nginx stop" --post-hook "service nginx reload" --quiet
```

# Tips

- 使用`site:blbana.cc`可以查看自己域名是否被收录

  ![https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/7.png](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/Google-Analytics/7.png)