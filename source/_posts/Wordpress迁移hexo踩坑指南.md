---
title: Wordpress迁移hexo踩坑指南
categories:
  - blogs
tags:
  - blogs
  - hexo
toc: true 文章目录
author: BlBana
date: 2016-12-05 11:24:48
description:
comments:
original:
permalink:
---

>最近看到了不少基于github pages的hexo上的blog，作为强迫症晚期的我，心动了...... 这意味着踩坑之路即将开始，刚好可以学习一下git，github的使用，顺便熟悉一下nodeJS这种Web开发网站的搭建方法，虽然道路曲折，但是收获颇多啊，现在来捋一捋这两天我踩的坑。

<!-- more -->
## Wordpress迁移hexo踩坑指南

**首先说说用hexo搭建个人博客的好处吧**
hexo是一款使用nodeJS开发的博客系统，一个markdown文件就是一篇blog，每次写好markdown以后，使用hexo编译成html文件，在同步到Github，Gitcafe这样的pages上，在自己的本地进行编写，我们可以随意的挑选喜欢的markdown编辑器，这里我推荐[马克飞象](https://maxiang.io/) 这款编辑器，界面简洁，写起blog很舒服呀。我是参考的[土神师父的blog进行搭建的](http://lorexxar.cn/2015/05/22/Hexo-blogs/) 。

### 安装Hexo

#### 安装git
寻找适合自己系统的git版本进行下载

#### 安装node.js
选择适合自己版本的node.JS进行安装，这里我安装的是最新版的node.JS

#### 安装Hexo
打开安装的git bash，输入命令
```bash
npm install -g hexo
```

#### 部署Hexo
找到合适的目录，在这个目录下打开git bash，执行以下的命令
```bash
hexo init
```
命令执行完后，在这个目录下会产生建立网站的文件。
搭建完毕以后
```bash
hexo g
hexo d
```
g命令的目的是将目录中的必要文件全部编译准备上传，会放在public目录下
d命令的目的是将准备好的文件同步到Github上去
>注意：要是想在本地搭建blog的话需要安装相应的插件，因为在Hexo 3.0后server部分被独立成了一个模块，否则无法在本地搭建。
```bash
hexo server -p 4000
```
在本地的4000端口建立了一个Web应用，我们可以访问[http://localhost:4000](http://localhost:4000) 来访问我们的Hexo。

### 配置Github-Pages
使用Hexo编译以后的文件都将成为html的静态文件，这样我们就可以把这些文件放在Github Pages上了，不同于Wordpress这种基于动态语言，Github Pages是无法解释PHP，JAVA的，因此hexo完全满足我们的条件。

#### 注册Github
[Github官网](http://github.com)

#### 添加项目
建立一个仓库，仓库的名字必须是**username.github.io**，只有这样的规定格式Github才可以识别，并且在设置中需要打开Github Pages的功能，在简单的配置后就可以访问http://username.github.io来进入属于自己的主页了。

#### 配置SSH密钥
1.打开git bash，检查本机密钥
```bash
cd ~/.ssh
```
如果提示：*No such file or directory*
说明我们是第一次使用Github，我们需要创建SSH密钥，和Github构成连接

2.我们输入以下命令
```bash
ssh-keygen -t rsa -C "Email@Email.com"
```
成功生成密钥。

3.SSH密钥生成结束以后，可以在 ~/.ssh/目录下生成私钥id_rsa和公钥id_rsa.pub这两个文件，我们将id_rsa.pub里面的内容复制到Github中。

4.测试链接Github的服务器，在Git Bash中输入
```bash
ssh -T git@github.com
```
如果第一次连接服务器的话，会显示让你输入Github的账号密码，输入后成功连接。

#### 设置git的账户信息
```bash
git config --global user.name "yourname"
git config --global user.email "youremail@youremail.com"
```

### 配置文件_config.yml
等我们配置好Hexo和Github以后，我们需要利用Hexo的配置文件将两者联系起来，打开Hexo根目录下的_config.yml文件，将文件最下方的内容改为：
```
deploy
	type:git
	repository:git@github.com:BlBana/BlBana.github.io.git
	branch:master
```
配置完毕后，就可以正常将两者联系起来。
>注意：node.JS对于配置文件要求严格，需要在：后面跟上一个空格，第一次搭建问题出在了这里，找了老半天才发现怎么回事，坑爹~

### 绑定自定义域名
1.进入自己的soure文件夹中新建一个CNAME的文件，然后用文本编辑器打开，在首行填入你需要的域名，例如：drops.blbana.cc，**注意：前面没有http://**，然后使用hexo g&&hexo d上传部署；
2.在域名提供商那里解析域名；
3.添加一个CNAME，主机记录写drops，后面的记录值是：yourname.github.io；
4.等待DNS解析刷新

### 配置主题
Hexo主题的配置是十分简单的，我们需要把主题放到themes目录下去，这是我使用的[主题](https://github.com/luuman/hexo-theme-spfk.git)，可以使用命令进行安装
```bash
git clone https://github.com/luuman/hexo-theme-spfk.git themes/spfk
```
**注意：我在安装主题的时候，是直接在Github中下载下来了主题的源代码，放在了themes文件下，由于文件名字不是spfk，导致了CSS等一些文件路径出错，整个页面变成了...**
安装完成后，根据自己的需要填写配置文件。

---
至此我们的blog基本已经搭建完毕了，现在我们可以熟悉一下常用的命令有哪些。
```bash
hexo new "postName"                     #新建文章
hexo new page "pageName"                #新建页面
hexo generate                           #生成静态页面至public目录
hexo server                             #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy                             #将.deploy目录部署到GitHub
```

### markdown语法
我们使用[马克飞象](https://maxiang.io/)这款编辑器的时候，按ctrl+/键会出来详细的markdown语法，使用markdown进行文章的编写，我们的排版真不是烦恼啦，哈哈~
我们可以修改*Hexo/scaffolds/_post.md*，来修改我们每次创建一篇新文章的模板，方便快捷。

### Wordpress迁移到Hexo
现在搭建好了我们的hexo，现在剩下的就是把以前站点的文章全部导入到新blog里啦，官方文档里给出的方法是：
首先，安装 `hexo-migrator-wordpress` 插件。
```bash
npm install hexo-migrator-wordpress --save
```
在 WordPress 仪表盘中导出数据(“Tools” → “Export” → “WordPress”)。
插件安装完成后，执行下列命令来迁移所有文章。source 可以是 WordPress 导出的文件路径或网址。
```bash
hexo migrate wordpress <source>
```
执行完命令后发现我们的文章已经编程了markdown文件，存放在了**hexo\source\\_posts**文件下，之后我们执行命令
```bash
hexo clean
hexo g
hexo d
```
将所有的文章转变成html文件并推送到Github上。
>在这里我遇到一个问题是，当时从Wordpress导过来的文章无法正常的编译通过，出现了报错，检查发现是其中的一篇文章markdown语法出现了错误，多加了一个“.”，很坑爹，所以在写文章的时候要记住使用正确的markdown语法。

---

### 注意事项
1.**ERROR Deployer not found:git**，我们需要安装插件
```bash
npm install hexo-deployer-git --save
```
2.我们在对文件修改的时候，需要在source文件夹下进行修改，因为每次编译是以source文件下的内容为模板进行编译的，只是在public中进行修改的话，我们第二次编译，之前的修改就会被覆盖掉。

3.我使用的这个主题是自带多说的，因此不需要太多的配置，只是需要在配置文件中打开此项功能，将domain选项改为自己的域名就ok了，详细见[Hexo 主题：SPFK](http://luuman.github.io/2015/12/27/Hexo/HexoTheme/)，可以自定义留言板的样式，cool~

4.[官方问题解答](https://hexo.io/zh-cn/docs/troubleshooting.html)

---
### 结语
强迫症难受啊，看了就想换了，blog真的是太折螣了，以后就用这个啦，结合**马克飞象**，大爱呀，以后再折腾blog剁手剁手，我已看破红尘，内容才是王道。