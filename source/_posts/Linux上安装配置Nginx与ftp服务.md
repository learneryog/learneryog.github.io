---
title: Linux上安装配置Nginx与ftp服务 
date: 2019-08-04 14:20:07 
author: 杨明
img: /img/Random-img/41.jpg
top: false 
cover: false  
toc: true 
tags: 
- 环境搭建
- Linux
- Nginx
- ftp
categories: 环境配置
summary: 
---

# Nginx安装

首先在[Nginx官网](http://nginx.org/en/download.html)下载稳定版本的Nginx安装包，并将安装包上传到Linux。
使用 `tar -zxvf nginx-1.16.0.tar.gz` 将压缩包解压。

接下来先安装Nginx所需的依赖：

- gcc
	安装nginx需要先将官网下载的源码进行编译，编译依赖gcc环境，如果没有gcc环境，需要安装gcc：`yum install gcc-c++` 
- PCRE
	PCRE(Perl Compatible Regular Expressions)是一个Perl库，包括 perl 兼容的正则表达式库。nginx的http模块使用pcre来解析正则表达式，所以需要在linux上安装pcre库。`yum install -y pcre pcre-devel`
注：pcre-devel是使用pcre开发的一个二次开发库。nginx也需要此库。
- zlib
	zlib库提供了很多种压缩和解压缩的方式，nginx使用zlib对http包的内容进行gzip，所以需要在linux上安装zlib库。
	`yum install -y zlib zlib-devel`
- openssl
	OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及SSL协议，并提供丰富的应用程序供测试或其它目的使用。
	nginx不仅支持http协议，还支持https（即在ssl协议上传输http），所以需要在linux安装openssl库。
    `yum install -y openssl openssl-devel`
	
然后进入刚刚解压的文件夹，执行以下命令：
```
[root@localhost nginx-1.16.0]# ./configure \
> --prefix=/usr/local/nginx \
> --pid-path=/var/run/nginx/nginx.pid \
> --lock-path=/var/lock/nginx.lock \
> --error-log-path=/var/log/nginx/error.log \
> --http-log-path=/var/log/nginx/access.log \
> --with-http_gzip_static_module \
> --http-client-body-temp-path=/var/temp/nginx/client \
> --http-proxy-temp-path=/var/temp/nginx/proxy \
> --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
> --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
> --http-scgi-temp-path=/var/temp/nginx/scgi
```
可以看到文件夹下多出一个Makefile，然后依次执行make指令和make install指令：
```
[root@localhost ~]# make
[root@localhost ~]# make install
```

在启动Nginx之前，先创建临时文件夹：
```
[root@localhost ~]# mkdir /var/temp
[root@localhost ~]# mkdir /var/temp/nginx
```
## 启动Nginx

进入Nginx的安装目录：/usr/local/nginx，可以看到有conf、sbin、html三个文件夹，其中sbin中存放着的就是Nginx的可执行文件：
```
[root@localhost ~]# cd /usr/local/nginx
[root@localhost ~]# cd sbin
[root@localhost ~]# ./nginx
```

## 停止Nginx

1. 快速停止：查找到Nginx的进程id然后使用kill命令强制杀死进程。
```
[root@localhost ~]# cd /usr/local/nginx/sbin
[root@localhost ~]# ./nginx -s stop
```

2. 完整停止：等待Nginx进程处理完任务再进行停止。
```
[root@localhost ~]# cd /usr/local/nginx/sbin
[root@localhost ~]# ./nginx -s quit
```

## 重启Nginx

1. 先停止再启动
```
[root@localhost ~]# cd /usr/local/nginx/sbin
[root@localhost ~]# ./nginx -s quit
[root@localhost ~]# ./nginx
```

2. 重新加载配置文件
```
[root@localhost ~]# cd /usr/local/nginx/sbin
[root@localhost ~]# ./nginx -s reload
```

## 测试Nginx

Nginx的默认端口是80，需要在iptables中将80端口设为开放，打开iptables文件并加入以下信息：
打开iptables文件：`[root@localhost nginx]# vim /etc/sysconfig/iptables`
在其中加入：`-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT`
如下：
![iptables](/img/post-img/19-8-4-1.png)
保存退出。
重启iptables:
`[root@localhost ~]# service iptables restart`

然后在主机浏览器通过ip+端口访问Nginx，比如：192.168.94.192:80，看到如下信息，说明Nginx配置成功：
![Nginx欢迎界面](/img/post-img/19-8-4-2.png)

## 设置Nginx开机自启动

这里使用编写shell脚本的方式设置：
`[root@localhost ~]# vim /etc/init.d/nginx`
输入如下脚本：
``` shell
#!/bin/bash
# nginx Startup script for the Nginx HTTP Server
# it is v.0.0.2 version.
# chkconfig: - 85 15
# description: Nginx is a high-performance web and proxy server.
#              It has a lot of features, but it's not for everyone.
# processname: nginx
# pidfile: /var/run/nginx.pid
# config: /usr/local/nginx/conf/nginx.conf
nginxd=/usr/local/nginx/sbin/nginx
nginx_config=/usr/local/nginx/conf/nginx.conf
nginx_pid=/var/run/nginx.pid
RETVAL=0
prog="nginx"
# Source function library.
. /etc/rc.d/init.d/functions
# Source networking configuration.
. /etc/sysconfig/network
# Check that networking is up.
[ ${NETWORKING} = "no" ] && exit 0
[ -x $nginxd ] || exit 0
# Start nginx daemons functions.
start() {
if [ -e $nginx_pid ];then
   echo "nginx already running...."
   exit 1
fi
   echo -n $"Starting $prog: "
   daemon $nginxd -c ${nginx_config}
   RETVAL=$?
   echo
   [ $RETVAL = 0 ] && touch /var/lock/subsys/nginx
   return $RETVAL
}
# Stop nginx daemons functions.
stop() {
        echo -n $"Stopping $prog: "
        killproc $nginxd
        RETVAL=$?
        echo
        [ $RETVAL = 0 ] && rm -f /var/lock/subsys/nginx /var/run/nginx.pid
}
# reload nginx service functions.
reload() {
    echo -n $"Reloading $prog: "
    #kill -HUP `cat ${nginx_pid}`
    killproc $nginxd -HUP
    RETVAL=$?
    echo
}
# See how we were called.
case "$1" in
start)
        start
        ;;
stop)
        stop
        ;;
reload)
        reload
        ;;
restart)
        stop
        start
        ;;
status)
        status $prog
        RETVAL=$?
        ;;
*)
        echo $"Usage: $prog {start|stop|restart|reload|status|help}"
        exit 1
esac
exit $RETVAL
```
保存并退出。

设置文件的访问权限：
`chmod a+x /etc/init.d/nginx` (a+x ==> all user can execute  所有用户可执行)

加入到rc.local中：
`[root@localhost ~]# vim /etc/rc.local`
加入一行：`/etc/init.d/nginx start`，下次重启会生效。

# ftp配置

## 安装vsftpd组件

`[root@localhost ~]# yum -y install vsftpd`
安装完后，有/etc/vsftpd/vsftpd.conf 文件，是vsftp的配置文件。

## 添加一个ftp用户

此用户就是用来登录ftp服务器用的。
`[root@localhost ~]#  useradd ftpuser`
添加密码：
`[root@localhost ~]# passwd ftpuser`

## 开启21端口

因为ftp默认的端口为21，而centos默认是没有开启的，所以要修改iptables文件
`[root@localhost ~]# vim /etc/sysconfig/iptables`
在22 -j ACCEPT 下面另起一行输入跟那行差不多的，只是把22换成21，然后：wq保存。
重启iptables:
`[root@localhost ~]# service iptables restart`

## 修改selinux

执行以下命令查看状态：
``` bash
[root@localhost ~]# getsebool -a | grep ftp  
allow_ftpd_anon_write --> off
allow_ftpd_full_access --> off
allow_ftpd_use_cifs --> off
allow_ftpd_use_nfs --> off
ftp_home_dir --> off
ftpd_connect_db --> off
ftpd_use_passive_mode --> off
httpd_enable_ftp_server --> off
tftp_anon_write --> off
```

执行上面命令，返回的结果中allow_ftpd_full_access和ftp_home_dir都是off，代表没有开启外网的访问
`[root@localhost ~]# setsebool -P allow_ftpd_full_access on`
`[root@localhost ~]# setsebool -P ftp_home_dir on`

## 关闭匿名访问

`[root@localhost ~]# vim /etc/vsftpd/vsftpd.conf`
![vsftptd.conf](/img/post-img/19-8-4-3.png)

## 设置开机自启

`[root@localhost ~]# chkconfig vsftpd on`

## 测试ftp服务

将Nginx作为项目图片服务器，需要配置nginx.conf文件：
![nginx.conf](/img/post-img/19-8-4-4.png)
使用java代码测试，需要依赖Apache的commons-net依赖。

``` java
package com.yangming.controller;

import org.apache.commons.net.ftp.FTP;
import org.apache.commons.net.ftp.FTPClient;
import org.junit.Test;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

public class FtpTest {
    @Test
    public void testFTPClient() throws IOException {
		//创建一个FTP客户端
        FTPClient ftpClient = new FTPClient();
		//创建连接
        ftpClient.connect("192.168.94.129",21);
        //登录
		ftpClient.login("ftpuser","ftpuser");
		//创建一个文件输入流，读取一张图片文件
        FileInputStream inputStream = new FileInputStream(new File(
                "C:\\Users\\Administor\\Pictures\\Saved Pictures\\timg.jpg"));
		//指定文件上传的路径
        ftpClient.changeWorkingDirectory("/home/ftpuser/www/images");
		//指定上传文件的格式，默认文本格式，需要改成二进制格式
        ftpClient.setFileType(FTP.BINARY_FILE_TYPE);
		//上传文件
        ftpClient.storeFile("test.jpg",inputStream);
        //退出登录
        ftpClient.logout();
    }
}
```
此时访问192.168.94.129/images/test.jpg就可以看到上传的图片了。
