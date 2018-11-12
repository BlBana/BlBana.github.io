---
title: Mac下Python连接MySQL
categories:
  - Python
tags:
  - Python
  - MySQL
toc: true 文章目录
author: BlBana
date: 2018-11-12 12:24:22
description:
comments:
original:
permalink:
---
> Mac和Windows下安装mysqlclient坑点记录

<!-- more -->

## Python连接MySQL（mysqlclient）
### Python连接MySQL类库

1. pymysql
2. python3：mysqlclient，python2：MySQLdb
3. MySQL官方：mysql-connector-python，mysql-connector-c

### Mac安装
1. `brew install mysql-connector-c`解决MySQL库的软连接问题，用于mysqlclient import libmysqlclient库
2. `brew install mysqlclient`安装mysqlclient库，用于Python连接MySQL
3. 安装`MySQL`，MySQL版本影响libmysqlclinent版本，mysqlclient版本依赖libmysqlclient版本

> Django依赖mysqlclient库连接MySQL
> 可以更改mysqlclient为pymysql
> 在__init__.py中加入代码：
```python
import pymysql
pymysql.install_as_MySQLdb()
```

## Windows安装mysqlclient

1. 安装MySQL客户端程序

2. 安装VC++ 14.0用于编译包，下载最新版本的Visual Studio即可

3. **mysql.h缺失**，Windows下采用wheel安装

   - `pip install wheel`
   - `pip install mysqlclient-1.3.13-cp37-cp37m-win_amd64.whl`

   > cp37表示python3.7，win_amd64表示64位python，或者32位python

---

### 问题
#### 链接mysql文件下的libmysqlclient库
`sudo ln -s /usr/local/mysql-8.0.13-macos10.14-x86_64/lib/libmysqlclient.21.dylib /usr/local/lib/libmysqlclient.21.dylib`
解决mysqlclient找不到库的问题
> 最新版本mysqlclient依赖.21.的库，而库版本依赖安装的MySQL版本，库可以向下兼容

#### 安装mysqlclient<13版本报错
1. brew install mysql-connector-c
2. 
```bash
cd /usr/local/Cellar/mysql-connector-c/6.1.11/bin/  
# 备份  
cp  mysql_config mysql_config.bak     
chmod u+w mysql_config  
vi mysql_config  
  
#   :114    找到第114行  
# 将  
#  libs="$libs -l "  
# 替换为  
#  libs="$libs -lmysqlclient -lssl -lcrypto"  
#保存  
```

#### 解决Library not loaded: @rpath/libmysqlclient.21.dylib
```bash
mdfind libmysqlclient | grep .21.
sudo ln -s /usr/local/mysql-8.0.13-macos10.14-x86_64/lib/libmysqlclient.21.dylib /usr/local/lib/libmysqlclient.21.dylib
```

#### 解决Library not loaded: libssl.1.0.0.dylib
```bash
mdfind  libssl | grep .1.0.0.
sudo ln -s /usr/local/mysql-8.0.13-macos10.14-x86_64/lib/libssl.1.0.0.dylib /usr/local/lib/libssl.1.0.0.dylib
```

#### 解决Library not loaded: libcrypto.1.0.0.dylib
```bash
mdfind libcrypto | grep .1.0.0.
sudo ln -s /usr/local/mysql-8.0.13-macos10.14-x86_64/lib/libcrypto.1.0.0.dylib /usr/local/lib/libcrypto.1.0.0.dylib
```

#### 参考连接
[安装mysqlclient报错](http://dushen.iteye.com/blog/2425130)
[libmysqlclient加载失败](https://stackoverflow.com/questions/4546698/library-not-loaded-libmysqlclient-16-dylib-error-when-trying-to-run-rails-serv/6100648#6100648)