---
title: PHP.ini安全配置
tags:
  - PHP
  - 服务器配置
  - 运维
id: 183
categories:
  - PHP
date: 2016-07-30 18:34:07
---

#### 提升PHP安全的配置

### 0x01：禁用远程url文件处理功能
<!--more-->
`
allow_url_fopen = off
`

### 0x02：禁用注册全局变量

`
register_globals = off
`

### 0x03：限制PHP的读写操作

修改本地文件的读写权限

`open_basedir = /var/www/files`

### 0x04：Posing Limit

限制PHP的执行时间，内存使用量，post和upload的数据

`
max_execution_time = 30
max_input_time = 60
memory_limit = 16M
upload_max_filesize = 2M
post_max_size = 8M
`

### 0x05：禁用错误消息和启用日志功能

`
safe_mode = off
safe_mode_gid = on
`

### 0x06：安全模式的配置

`
safe_mode = off
safe_mode_gid = on
`

在默认情况下，可以将php配置为安全模式，在这种模式下Apache禁止访问文件，环境变量和二进制程序，所以打开安全用户组，在这个用户组内的成员可以正常进行访问。

将二进制文件存放在一个目录中

`
safe_mode_exec_dir = /var/www/base
`

### 0x07：系统函数的禁用

`disable_funtion=`

禁用一些危害较大的系统函数