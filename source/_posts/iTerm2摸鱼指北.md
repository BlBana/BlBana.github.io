---
title: iTerm2摸鱼指北
categories:
  - Tools
tags:
  - Tools
toc: true 文章目录
author: BlBana
date: 2019-10-12 15:50:40
description:
comments:
original:
permalink:
---

> 作为一个肥宅程序员，天天都要用到Terminal，怎么能不好好折腾折腾iTerm2这么一个工作利器呢，话不多说，直接走起，让你们见识一些我的小GaiGai ~ 记录一下我这爱折腾的成果

<!-- more -->

---

# 1. 介绍

自2017年开始用Macbook，终端一直用的是iTerm2，每次拿到新机都是先折腾这个，前前后后也用的比较熟练了，给大家分享一下我的一些iTerm2的使用和配置。

## 1.1 iTerm2介绍

iTerm2是一款完全免费，为MacOS打造的一款终端工具，堪称终端利器，程序员必备。

下载链接：[https://www.iterm2.com/](https://www.iterm2.com/)

## 1.2 效果图

话不多说，先上我终端效果图好吧：

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/iTerm2/iTerm2.png)

> 怎么样，我的小GaiGai是不是很漂亮 ~ 下面看下详细的配置

# 2. 配置走起

## 2.1 oh my zsh

> iTerm2这么个利器，当然要配置功能强大的shell了，原生zsh配置起来比较容易掉头发，但是已经有了开源的配置oh my zsh提供给我们，只需要简单配置就能随意的更改样式，插件等功能。

### 2.1.1 安装

```bash
# curl
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

# wget
sh -c "$(wget -O- https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

安装结束后会自动把shell切换到zsh上，如果你机子上没有安装，可以先把zsh安装上。

```bash
brew install zsh
```

之后会在用户根目录下生成`.zshrc`文件和`.oh-my-zsh`，前者是配置文件，后者是存放`themes，plugins`的文件夹。主要还是配置文件，有需要的时候可以安装自定义的插件到`.oh-my-zsh`目录。

### 2.1.2 常用插件

#### 插件推荐

这里推荐一下我比较常用的几个插件：

```bash
plugins=(git z zsh-syntax-highlighting zsh-autosuggestions)
```

- **Git**：用于在你的主机名后显示git项目信息，比如分支，目录，当前项目状态等信息，可以使用各种git命令缩写；
- **z**：用于目录间快速跳转，比如之前进入过`~/User/my_project`目录后，下一次再想进入的时候，直接`z my_project`即可，对于较长目录跳转非常的实用；
- **zsh-syntax-highlighting**：用于高亮显示常见的命令，比如ls，cd等命令为绿色，输入错误命令时会显示红色；
- **zsh-autosuggestions**：当你输入命令的时候，会用灰色显示出你可能想输入的推荐命令，直接键盘`→`就能补全命令，效率神器。

这些插件基本上已经oh my zsh自带了，直接在配置文件里找到**plugins**参数在括号里写入想要的插件即可。

#### 自定义插件安装

有时候上述几个插件默认没有安装，或者你需要安装第三方插件，自己写插件，可以手动安装，并进行配置。

```bash
~/.oh-my-zsh/custom/plugins
~/.oh-my-zsh/custom/themes
```

下载并将插件文件或者主题文件放到上述两个文件夹中，配置文件添加即可。

- [github插件列表](https://github.com/robbyrussell/oh-my-zsh/tree/master/plugins)

### 2.1.3 终端样式

```bash
ZSH_THEME="ys" # 配置.zshrc
```

可以在线预览各种主题样式，调个自己喜欢的更改配置即可。没有安装的主题，按照插件安装方式安装即可。

- [github主题列表](https://github.com/robbyrussell/oh-my-zsh/wiki/Themes)



> 整个oh my zsh Shell环境配置没有问题了，接下来看看iTerm2配置。

## 2.2 iTerm2配置

### 2.2.1 Window配置

配置沉浸式边框，修改默认`Theme`为`Minimal`

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/iTerm2/window.png)

### 2.2.2 主题配置

在我效果图里使用的是，**Dracula**配色，可以自行下载进行配置。

- [Dracula下载地址](https://draculatheme.com/iterm/)

解压后导入配置到iTerm2中，红框处选择`import`导入配置文件`Dracula.itermcolors`。选择Dracula更换配色。

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/iTerm2/color.png)

### 2.2.3 字体配置

我这里字体使用的是M+等宽字体，[下载链接在此](https://mplus-fonts.osdn.jp/)，下载后安装到电脑上。根据需求选择字体，配置粗细，大小等设置。

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/iTerm2/font.png)

### 2.2.4 背景配置

#### 透明度

首先是透明度的配置。在A框处的进度条可以设置透明度，顺便提一下在下面`Settings for New Windows`可以设置默认窗框长款。

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/iTerm2/background.png)

#### 肥宅快乐小GaiGai

接下来就是激动人心的环节了，小GaiGai ！！！如上图的B框处勾选启动背景图片，导入你喜欢的小姐姐即可，Mode处可以设置图片的平铺方式。**那么问题来了，哪里能找到漂亮的小姐姐呢，送上大佬们一记壁纸Site。**

- [The best wallpapers on the Net!](https://wallhaven.cc/)

我的小姐姐都是从这里来的，有图片热榜，分类，图片大小调整，壁纸试用等等功能，肯定能找到你心仪的小GaiGai ~

### 2.2.5 Status Bar配置

接下来就是个高逼格的东西了，配置终端的状态栏。可以拉取电量信息，CPU使用率，内存使用率，网络监控，时钟等功能，真乃是装逼利器。

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/iTerm2/status1.png)

![](https://blog-img-1252112827.cos.ap-chengdu.myqcloud.com/image/jpg/iTerm2/status2.png)

> 目前能想到的配置就是上面这些了，以后有什么新发现再补充上来。

# 3. 使用技巧

### 3.1 常用快捷键 

推荐几个常用快捷键吧：

- command + d ：垂直分割窗口
- command + shift + d：水平分割窗口

好了，结束，我就用这两个 。。。

# 4. 收工

用了两年多的iTerm2，今天终于抽时间整理了已经配置方法，现在云端保存了一份配置文件，以后也懒得折腾了。

> 菜鸡打滚，溜了 ~

