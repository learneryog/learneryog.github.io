---
title: Ubuntu环境下安装配置Docker
date: 2019-05-30 14:55:33
cover: /img/Random-img/75.jpg
categories:
- 环境配置
tags:
- Linux
- Ubuntu
- 环境搭建
- docker
---

> 安装环境：Ubuntu 18.04

### 1. 系统要求

Ubuntu系统对Docker的支持十分成熟，只要是64位的即可。
Docker目前支持的最低的Ubuntu版本为14.04 LTS，但实际上从稳定性上考虑，推荐使用16.04 LTS或18.04 LTS版本，并且系统内核越新越好，以支持Docker最新的特性。
> 查看内核版本详细信息：（二选其一即可）
> $ uname -a       
> $ cat /proc/version

### 2. 添加镜像源

首先需要安装apt-transport-https等软件包支持https协议的源：
> $ sudo apt-get update
> $ sudo apt-get install \
>    apt-transport-https \
>    ca-certificates \
>    curl \
>    software-properties-common

添加源的gpg密钥：
> $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-get add -
> OK

确认公钥：
> $ sudo apt-key fingerprint 0EBFCD88

获取当前操作系统的代号：
> $ lsb_release -cs
> bionic

添加Docker稳定版的官方软件源：(注意“bionic”处与查看到的代号相一致)
> $ sudo add-apt-erpository \
> "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
> bionic stable"

添加成功后，再次更新apt软件包缓存：
> $ sudo apt-get update

### 3. 开始安装Docker

添加完源之后就可以安装最新版的Docker了，软件包名称为docker-ce，代表是社区版本：
> $ sudo apt-get install -y docker-ce

如果系统中存在较旧版本的Docker，会提示是否先删除，选择是即可。
安装成功后，会自动启动Docker服务。
查看docker信息：
> sudo docker version

![docker信息](/img/post-img/19-5-30-1.png)

### 4. 配置Docker服务

为了避免每次使用Docker命令时都需要切换到特权身份，可以将当前用户加入安装中自动创建的docker用户组：(USER_NAME部分为当前用户名)
> sudo username -aG docker USER_NAME

用户更新组信息，退出并重新登录后即可生效。

Docker服务启动实际上是调用了dockerd命令，支持多种启动参数。因此，用户可以直接通过执行dockerd命令来启动Docker服务，如下面的命令启动Docker服务，开启debug模式，并监听在本地的2376端口：
> $ dockerd -D -H tcp://127.0.0.1:2376

这些选项可以写入/etc/docker/路径下的daemon.json文件中，由dockerd服务启动时读取：
> {
> &emsp;&emsp;"debug": true,
> &emsp;&emsp;"hosts": ["tcp://127.0.0.1:2376"]
> }

当然，操作系统也对Docker服务进行了封装，以Ubuntu为例，Docker服务的默认配置文件为/etc/default/docker，可以通过修改其中的DOCKER_OPTS来修改服务启动的参数，例如让Docker服务开启网络2375端口的监听：
> DOCKER_OPTS="$DOCKER_OPTS -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"

修改之后，通过service命令来重启Docker服务：
> sudo service docker restart

另外，使用国外的镜像源下载速度慢，修改镜像源可以加快下载速度。切换到root权限，打开/etc/docker/daemon.json文件，添加如下内容：
> {
"registry-mirrors": [
"https://kfwkfulq.mirror.aliyuncs.com",
"https://2lqq34jg.mirror.aliyuncs.com",
"https://pee6w651.mirror.aliyuncs.com",
"https://registry.docker-cn.com",
"http://hub-mirror.c.163.com"
],
"dns": ["8.8.8.8","8.8.4.4"]
}

此外，如果服务不正常，可以通过查看Docker服务的日志信息来解决问题，Ubuntu系统上可以执行`journalctl -u docker.service`

每次重启Docker服务后，可以通过查看docker信息（`docker info`命令），确保服务已经正常运行。