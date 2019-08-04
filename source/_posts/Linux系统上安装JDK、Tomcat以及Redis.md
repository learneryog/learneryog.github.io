---
title: Linux系统上安装JDK、Tomcat以及Redis
date: 2019-05-19 11:24:59
cover: /img/Random-img/107.jpg
categories: 
   - 技术文章
tags: 
   - Linux
   - 环境搭建
   - redis
---

> 环境：VMware搭载CentOS6.5版本Linux系统，SecureCRT远程登录控制，安装JDK1.8，Tomcat9.0.10，Redis5.0.4

### 一、安装JDK1.8

首先检查Linux系统上是否有JDK，一般Linux系统会有默认的openJDK，将其卸载掉。

``` javascript
rpm -qa | grep -i java     // 查询系统上是否存在默认JDK
 
rpm -e --nodeps 查出来的程序名     // 将查询出来的默认JDK卸载掉
```

安装依赖：

`
yum install glibc.i686
`

将下载好的.tar.gz压缩包上传到Linux系统，解压到/usr/local/java目录下。

``` javascript
mkdir -p /usr/local/java      // 循环创建文件夹
 
tar -zxvf jdk-8u181-linux-x64.tar.gz -C /usr/local/java     // 将压缩包解压到指定目录下
```

配置环境变量:

``` javascript
vim /etc/profile     // 使用vim编辑器查看配置文件
// 按下i键进入修改模式
//在文件末尾加入以下内容：

#set java enviroment
JAVA_HOME=/usr/local/java/jdk1.8.0_181
CLASSPATH=.:$JAVA_HOME/lib.tools.jar
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME CLASSPATH PATH

// 按Ctrl+C退出编辑，:wq保存退出
// 执行以下命令重新加载环境变量

source /etc/profile      

```

至此JDK1.8安装完成。

### 二、安装Tomcat9.0.10

将下载好的.tar.gz压缩包上传到Linux系统，并解压到/usr/local/tomcat目录下：

``` javascript
mkdir -p /usr/local/tomcat     // 创建文件夹
 
tar -zxvf apache-tomcat-9.0.10.tar.gz -C /usr/local/tomcat    // 解压到指定文件夹
```

进入安装目录下的bin目录，运行startup.sh文件，启动服务器。

``` javascript
cd /usr/local/tomcat/apache-tomcat-9.0.10.tar.gz/bin
 
./startup.sh       // 启动服务
```

看到如下信息表示服务启动成功：

![tomcat启动成功](/img/post-img/5-19.png)

在主机（安装VMware的电脑）上访问ip（Linux虚拟机的ip）+端口号（8080），咦，怎么访问不到？

原来是Linux防火墙默认拦截了8080端口，只要把端口打开就好了。

``` javascript
/sbin/iptables -I INPUT -p tcp --dprot 8080 -j ACCEPT    // 打开8080端口
 
/etc/rc.d/init.d/iptables save     // 保存配置
```

然后再访问ip+8080就可以看到熟悉的tom猫页面了：

![tomcat主页](/img/post-img/5-19-1.png)

关闭服务器：

``` javascript
// 在bin目录下执行：
./shutdowm.sh
```

至此Tomcat9.0.10安装完毕。

### 三、安装Redis5.0.4

安装依赖：

`
yum install gcc-c++
`

到官网下载压缩包，上传到Linux系统，并解压。

`
tar -zxvf redis-5.0.4.tar.gz    // 直接解压到当前目录即可
`

进入刚刚解压的redis-5.0.4目录，在该目录下执行   make   命令，进行编译。

等一会过后出现如下信息表示编译成功：

![redis编译成功](/img/post-img/5-19-2.png)

然后在该目录下执行以下命令进行安装：（/usr/local/redis文件夹会自动创建）

``` javascript
[root@localhost redis-5.0.4]# make PREFIX=/usr/local/redis install

// 将配置文件复制到安装目录下，用以自定义配置
[root@localhost redis-5.0.4]# cp redis.conf /usr/local/redis
```

前端启动redis服务：

``` javascript
[root@localhost redis-5.0.4]# cd /usr/local/redis
[root@localhost redis]# ./bin/redis-server
```

出现如下信息表示服务启动成功：

![redis启动成功](/img/post-img/5-19-3.png)

其中6379是Redis默认端口号，PID是进程ID，方便停止进程。

在另一个窗口（session）中使用如下命令连接Redis服务：

``` javascript
// 默认连接本机的6379端口
[root@localhost bin]# ./redis-cli      

// 根据ip和端口号连接指定redis服务
[root@localhost bin]# ./redis-cli -h 127.0.0.1 -p 6379   
```

后端启动redis服务：

进入redis.conf文件修改daemonize属性为yes

然后启动服务的同时加载配置文件。

`
[root@localhost redis]# ./bin/redis-server ./redis.conf
`

页面显示没有前端启动那么花里胡哨，但确实启动了，所谓后端启动。

关闭redis：

``` javascript
// 1、查询到pid   使用   kill -9 pid  杀死进程
[root@localhost redis]# ps -ef | grep redis
[root@localhost redis]# kill -9 pid

// 2、正常结束
[root@localhost redis]# ./bin/redis-cli shutdown
```

至此，Redis5.0.4安装完毕。

开启你的Redis之旅吧。