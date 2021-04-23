---
title: "Mysql ERROR When Starting!"
date: 2020-03-11 01:55:05
tags: [Mysql, Error]
category: [Debug]
---

**错误关键句**：
1. .dyld: Library not loaded: /usr/local/opt/openssl/lib/libssl.1.0.0.dylib
2. ERROR! The server quit without updating PID file

**解决办法简述**
```
$ brew upgrade 
```
<!-- more -->
**详细错误提示**
```
dyld: Library not loaded: /usr/local/opt/openssl/lib/libssl.1.0.0.dylib
  Referenced from: /usr/local/Cellar/mysql@5.7/5.7.24/bin/my_print_defaults
  Reason: image not found
Starting MySQL
.dyld: Library not loaded: /usr/local/opt/openssl/lib/libssl.1.0.0.dylib
  Referenced from: /usr/local/Cellar/mysql@5.7/5.7.24/bin/my_print_defaults
  Reason: image not found
dyld: Library not loaded: /usr/local/opt/openssl/lib/libssl.1.0.0.dylib
  Referenced from: /usr/local/Cellar/mysql@5.7/5.7.24/bin/my_print_defaults
  Reason: image not found
/usr/local/Cellar/mysql@5.7/5.7.24/bin/mysqld_safe: line 198:  2483 Abort trap: 6           nohup /usr/local/Cellar/mysql\@5.7/5.7.24/bin/mysqld --basedir=/usr/local/Cellar/mysql\@5.7/5.7.24 --datadir=/usr/local/var/mysql --plugin-dir=/usr/local/Cellar/mysql\@5.7/5.7.24/lib/plugin --user=mysql --log-error=zhiruis-MacBook-Pro.local.err --pid-file=/usr/local/var/mysql/zhiruis-MacBook-Pro.local.pid < /dev/null > /dev/null 2>&1
 ERROR! The server quit without updating PID file (/usr/local/var/mysql/zhiruis-MacBook-Pro.local.pid).
 ```
 ------- 
 ## 过程:

 **背景介绍**（一些废话）
 今天想 dump 数据库的时候突然报了无法找到 mysql 的对应版本号的错误，我非常的奇怪，因为之前用的都好好的。本着重启解决一切的原则，我重启了 mysql 服务和电脑，结果报了如上的错误。重启无果，我决定面对现实来解决这个突如其来的问题。
 
 **排查逻辑**
1. 先重启再说。（上文）
2. 科学上网 + CSND。参考了几篇前辈的文章：https://blog.csdn.net/super_man_ww/article/details/51460572     
https://blog.csdn.net/abs1004/article/details/84839894
    虽然说报错很像，但其根本原因与我相差甚远，但他们为我提供了很棒的解决思路（就是别想着靠别人，仔细研究报错提示来解决问题。）
3. 可能性思考。结合报错信息（"ssl library not found"）我回顾了近期操作是否影响了 mysql 和 ssl 的相关配置。最后定位到我昨天因为 python package 的依赖需要，升级了 openssl 版本。根据这一信息，我开始尝试解决。↓

**具体步骤**
 1. 我根据路径 /usr/local/opt/openssl/lib/ 查看引起报错的源文件。果不其然，其同名文件版本为1.1，而不是1.0。不同的文件名导致 mysql 无法找到该文件。![图片1](https://img-blog.csdnimg.cn/20200310170004396.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODM1NDk2,size_16,color_FFFFFF,t_70)
 2. 参考 https://stackoverflow.com/questions/59006602/dyld-library-not-loaded-usr-local-opt-openssl-lib-libssl-1-0-0-dylib 后，我发现是 mac 系统动态库的问题。通过输出 mysql 5.7.24 所依赖的动态链接库，显示 mysql 5.7.24 与 libssl.1.0.0 版本兼容（而我昨天升到了 1.1）。
```
$ otool -L /usr/local/Cellar/mysql@5.7/5.7.24/lib/libmysqlclient.dylib
/usr/local/Cellar/mysql@5.7/5.7.24/lib/libmysqlclient.dylib:
	/usr/local/opt/mysql@5.7/lib/libmysqlclient.20.dylib (compatibility version 20.0.0, current version 20.0.0)
	/usr/local/opt/openssl/lib/libssl.1.0.0.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/local/opt/openssl/lib/libcrypto.1.0.0.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 400.9.4)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.200.5)
```
 3. 所以假设的解决方法有两种：
	1. 回退 libssl 版本到 1.0.0 。
	2. 升级 mysql 到更新的版本看是否兼容 libssl 1.1
 4. 我更倾向于后者，于是...  "brew upgrade" 开始进入漫长的更新过程。
```
$ brew upgrade 
```
 5. N 小时过后，我成功的再次登陆 mysql。感动！
 ```
 $ mysql.server start
Starting MySQL
.. SUCCESS!
```
