---
title: 使用Jedis远程连接Redis时的小插曲
date: 2019-05-19 16:58:23
cover: /img/Random-img/78.jpg
categories:
   - 技术文章
tags:
   - Jedis
   - redis
---

> Jedis是远程连接redis的主流集成工具，在使用Jedis的过程中踩了几个坑，特此纪念。

从Maven依赖库库中下载两个jar包，分别是commons-pool2-2.4.2.jar和jedis-2.9.0.jar，版本不作要求。将这个两个jar包导入到工程中，然后开始编写程序。

先写一个简单的测试用例：
![测试用例](/img/post-img/19-5-19-13.png)

其中192.168.94.129是我Linux虚拟机的ip地址，在保确保虚拟机上开启redis服务的前提下，运行测试用例，发现连接失败，怎么回事？

原来又是Linux防火墙，Linux防火墙将6379端口拦截掉了，那我们就在Linux系统上将6379端口打开：

``` javascript
[root@localhost redis]# /sbin/iptables -I INPUT -p tcp --dport 6379 -j ACCEPT
[root@localhost redis]# /etc/rc.d/init.d/iptables save
```

然后再运行一次测试用例，发现和刚才一样，还是连接超时，一大堆的异常，这又是怎么回事呢？端口已经打开了呀！

可是仔细观察就会发现，在Linux虚拟机上连接到Redis服务的时候显示是127.0.0.1：6379>，那我们把ip换成127.0.0.1试一下，很遗憾，失败了。

是不是配置文件搞的鬼呢？进入redis.conf文件看一下，果然！有这么一段话：
![redis.conf](/img/post-img/19-5-19-14.png)

> bind后边指明的ip地址才是访问redis的合法地址，所以我们在其下边加入bind 192.168.94.129之后保存退出。

此时我们重新启动redis服务：

``` javascript
[root@localhost redis]# ./bin/redis-cli shutdown
[root@localhost redis]# ./bin/redis-server ./redis.conf 
```

然后再运行一次测试代码，哇，一抹绿色终于出现了，终于连接成功，可以用Java代码来操作redis啦，redis有什么指令，Jedis就有什么方法，所以Jedis的API根本不用去记，只要知道Redis有哪些常用的指令就好啦！

OK，问题解决啦，继续你的旅程吧！加油。