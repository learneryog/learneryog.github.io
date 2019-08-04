---
title: Docker基础（一）
date: 2019-05-31 16:31:36
cover: img/Random-img/83.jpg
categories: 
- Dcoker
tags:
- Docker
---

> 今天开始正式学习Docker，边学习边总结，任重而道远。

- 启动docker服务：systemctl start docker
- 启动守护进程：systemctl daemon-reload
- 重启docker服务：systemctl restart docker
- 关闭docker服务：systemctl stop docker

[toc]
## 一、Docker镜像
### 1. 获取镜像
格式：`docker [image] pull NAME[:TAG]`
例如：获取一个Ubuntu18.04系统的基础镜像：$ docker pull ubuntu:18.04
如果不显式指定TAG，则默认会选择latest标签，image指定镜像源，一般不用
pull子命令支持的选项：
- -a，--all-tags=true | false ：是否获取仓库中的所有镜像，默认为否；
-  --disable-content-trust ：取消镜像的内容校验，默认为真。

下载镜像到本地后就可以随时使用该镜像了，例如利用该镜像创建一个容器，在其中运行bash应用，打印“Hello World”：
![使用镜像创建容器](/img/post-img/19-6-1-1.png)

### 2. 查看镜像信息

#### 2.1、使用images命令列出镜像
格式：`docker images`或者`docker image ls`
在列出的信息中，可以看到几个字段：
> - REPOSITORY：来源于哪个仓库，比如ubuntu表示ubuntu系列的基础镜像；
> - TAG：镜像的标签信息，比如18.04、latest表示不同的版本信息。一般用来标记来自同一仓库的不同镜像。标签只是标记，并不能标识镜像内容；
> - IMAGE ID：镜像的ID（唯一标识），如果两个镜像的ID相同，说明它们实际上指向了同一个镜像，只是具有不同的标签而已；
> - CREATED：创建时间，镜像最后更新的时间；
> - SIZE：镜像大小，优秀的镜像往往体积都较小。只表示该镜像的逻辑大小，实际上相同的镜像在本地只会存储一份。

images子命令支持的选项：
- a，--all=true | false：列出所有（包括临时文件）镜像文件，默认为否；
- --digests=trur | false：列出镜像的数字摘要值，默认为否；
- -f，--filter=[]：过滤列出的镜像，如dangling=true只显示没有被使用的镜像；也可以指定带有特殊标注的镜像等；
- --format="TEMPLATE"：控制输出格式，如.ID代表ID信息，.Repository代表仓库信息等；
- --no-trunc=true | false：对输出结果中太长的部分是否进行截断，如ID信息，默认为是；
- -q，--quiet=true | false：仅输出ID信息，默认为否。

更多子命令可通过`man docker-images`来查看。

#### 2.2、使用tag命令添加镜像标签
格式：docker tag 旧标签 新标签
例如：添加一个新的myubuntu:latest镜像标签：
`$ docker tag ubuntu:latest myubuntu:latest`
再次使用docker images，可以看到多了一个myubuntu:latest标签的镜像，之后，用户就可以用新的标签来使用这个镜像了。

#### 2.3、使用inspect命令查看详细信息
格式：docker [image] inspect 镜像标签
例如：`docker [image] inspect ubuntu:18.04`
结果返回一个JSON格式的消息，如果只要其中的一项内容，可以使用-f来指定。例如获取镜像的Id:
> $ docker inspect -f {{".Id"}} ubuntu:18.04
sha256:7698f282e5242af2b9d2291458d4e425c75b25b0008c1e058d66b717b4c06fa9

#### 2.4、使用history命令查看镜像历史
格式：docker history 镜像标签
例如：查看ubuntu:18.04镜像的创建过程：
`$ docker history ubuntu:18.04`

### 3. 搜寻镜像
格式：docker search [option] keyword
支持的命令选项主要包括：
- -f，--filter filter：过滤输出内容
- --format string：格式化输出内容
- --limit int：限制输出结果个数，默认为25个
- --no-trunc：不截断输出结果

例如：搜索官方提供的带nginx关键字的镜像：
`$ docker search --filter=is-official=true nginx`
搜索所有收藏数超过4的、关键词包括tensorflow的镜像：
`$ docker search --filter=stars=4 tensorflow`

### 4. 删除和清理镜像
#### 4.1、使用标签删除镜像
格式：`docker rmi IMAGE [IMAGE...]`或者`docker image rm IMAGE [IMAGE...]`
支持选项包括：
- -f，-force：强制删除镜像，即使有容器依赖它；
- -no-prune：不要清理未带标签的额父镜像。

例如：删除myubuntu:latest镜像：
`$ docker rmi myubuntu:latest`

#### 4.2、使用镜像ID来删除镜像 
格式：`docker rmi IMAGE-ID [IMAGE-ID...]`
会先尝试删除所有指向该镜像的标签，然后删除该镜像文件本身。注意，当有该镜像创建的容器存在时，镜像文件默认是无法删除的（**`docker ps -a`查看本机所有容器**）。如果要强行删除，可以用 -f 选项，但是不推荐使用，正确的做法是先删除依赖该镜像的所有容器，再来删除镜像（**`docker rm 容器` 删除指定容器**）。

#### 4.3、清理镜像
使用docker一段时间后，系统中可能会遗留一些临时的镜像文件，以及一些没有被使用的镜像，这时用到镜像清理命令。
格式：docker image prune
支持的选项包括：
- -a，--all：删除所有无用镜像，不光是临时文件；
- -filter filter：只清除符合给定过滤器的镜像；
- -f，-force：强制删除镜像，而不进行提示确认。

### 5. 创建镜像
#### 5.1、基于已有容器创建
格式：docker [container] commit [OPTIONS] CONTAINER [REPOSITORY [:TAG]]
主要选项包括：
- -a，--author=""：作者信息；
- -c，--change=[]：提交的时候执行dockerfile指令，包括CMD | ENTRYPOINT | ENV | EXPOSE | LABEL | ONBUILD | USER | VOLUME | WORKDIR等；
- -m，--message=""：提交消息；
- -p，--pause=true：提交时暂停容器运行。

#### 5.2、基于本地模板导入
格式：docker [image] import [OPTIONS] file|URL| - [REPOSITORY[:TAG]]
通过下载[OpenVz](http://openvz.org/Download/templates/precreated)提供的模板压缩包导入：
`$ cat ubuntu-18.04-x86_64-minimal.tag.gz | docker import - ubuntu:18.04`

#### 5.3、基于Dockerfile创建
Dockerfile是一个文本文件，利用给定的指令描述基于某个父镜像创建新镜像的过程。

### 6. 存出和载入镜像
#### 6.1、存出镜像
格式：docker [image] save
该命令支持 -o、-output string参数，导出镜像到指定的文件中。
例如，导出本地的ubuntu:18.04镜像为文件ubuntu_18.04.tar：
`$ docker save -o ubuntu_18.04.tar ubuntu:18.04`
之后用户就可以通过复制ubuntu_18.04.tar文件将该镜像分享给他人。
#### 6.2、载入镜像
格式：docker [image] load
支持 -i、-input string选项，从指定文件中读入镜像内容。
例如，从文件ubuntu_18.04.tar导入镜像到本地镜像列表：
`$ docker load -i ubuntu_18.04.tar`
### 7. 上传镜像
格式：docker [image] push NAME[:TAG] | [REGISTRY_HOST[:REGISTRY_POST] / ] NAME [:TAG]
第一次上传时会提示输入登录信息或进行注册，之后登录信息会记录到本地的~/.docker目录下。

## 二、Docker容器
### 1. 创建容器
#### 1.1、新建容器
格式：docker [container] create
> $ docker create -it ubuntu:latest
3fa6a10fb9243158d5673430f1930ef5b4b8af44875f9822a229d2442a9c415c

新建的容器处于停止状态，可以使用`docker [container] start`命令来启动它。
由于容器是整个Docker技术栈的核心，create命令支持的选项十分复杂。选项主要包括以下几大类：与容器运行模式相关、与容器环境配置相关、与容器资源限制和安全保护相关。
![与容器运行模式相关](/img/post-img/19-6-1-2.jpg)
![其他选项](/img/post-img/19-6-1-3.jpg)
![续](/img/post-img/19-6-1-4.jpg)

#### 1.2、启动容器
格式：docker [container] start CONTAINER ID
可以通过`docker ps`命令查看运行中的容器

#### 1.3、创建并启动容器
格式：docker [container] run
等价于先执行docker create命令，再执行docker start命令。
> $ docker run ubuntu /bin/echo 'Hello World'
Hello World

利用run命令来创建并启动容器时，Docker在后台运行的标准操作包括：
 - 检查本地是否存在指定的镜像，不存在就从公有仓库下载；
 - 利用镜像创建一个容器，并启动该容器；
 - 分配一个文件系统给容器，并在只读的镜像层外面挂载一层可读写层；
 - 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去；
 - 从网桥的地址池配置一个IP地址给容器；
 - 执行用户指定的应用程序；
 - 执行完毕后容器被自动终止。

#### 1.4、守护态运行
通过添加 -d 参数来实现容器在后台以守护态形式运行。
> $ docker run -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
171067c30ef818f79d82f173331e59e9ce2297857106c3fface447a90aa81fd3

容器启动后会返回一个唯一的id，通过`docker ps`或`docker container ls`来查看容器信息。

#### 1.5、查看容器输出
格式：docker [container] logs
支持的选项：
- -details：打印详细信息；
- -f，-follow：持续保持输出；
- -since string：输出从某个时间开始的日志；
- -tail string：输出最近的若干日志；
- -t，-timestamps：显示时间戳信息；
- -until string：输出某个时间之前的日志。

### 2. 停止容器
#### 2.1、暂停容器
格式：docker [container] pause CONTAINER [CONTAINER...]
处于暂停状态的容器，可以使用`docker [container] unpause CONTAINER [CONTAINER...]` 命令来恢复到运行状态。

#### 2.2、终止容器
格式：docker [container] stop [-t | --time [=10]] [CONTAINER...]
该命令会首先向容器发送SIGTERM信号，等待一段超时时间（默认为10秒）后，再发送SIGKILL信号来终止容器。
还可以通过`docker [container] kill` 直接发送SIGKILL信号来强行终止容器。
此时可以通过`docker container prune` 命令清除掉所有处于停止状态的容器。
处于终止状态的容器，可以通过`docker [container] start` 命令来重新启动。
`docker [container] restart` 命令会将一个运行态的容器先终止，然后再重新启动。

### 3. 进入容器
在使用 -d 参数时，容器启动后会进入后台，用户无法看到容器中的信息，也无法进行操作，这个时候如果需要进入容器操作，就需要用到此命令。
#### 3.1、attach命令
格式：docker [container] attach
支持三个主要选项：
- --detach-keys[=[]]：指定退出attach模式的快捷键序列，默认是CTRL-p，CTRL-q；
- --no-stdin=true | false：是否关闭标准输入，默认是保持打开；
- --sig-proxy=true | false：是否代理收到的系统信号给应用进程，默认是true。

然而使用attach命令有时候并不方便。当多个窗口同时attach到同一个容器的时候，所有窗口都会同步显示；当某个窗口因命令阻塞时，其他窗口也无法执行操作了。

#### 3.2、exec命令
格式：docker [container] exec
比较重要的参数有：
- -d：在容器中后台执行命令；
- --detach-keys=""：指定将容器切回后台的按键；
- -e：指定环境变量列表；
- -i：打开标准输入接受用户输入命令，默认值为false；
- --privileged=true | false：是否执行命令以高权限，默认值为false；
- -t：分配伪终端，默认值为false；
- -u：执行命令的用户名或ID

### 4. 删除容器
格式：docker [container] rm
主要支持的选项：
- -f：是否强行终止并删除一个运行中的容器；
- -l：删除容器的连接，但保留容器；
- -v：删除容器挂载的数据卷。

### 5. 导入和导出容器
容器的导入导出是为了实现容器从一个系统迁移到另外 一个系统。
导出容器：`docker [container] export`   -o 参数指定导出的tar文件名
导入容器：`docker [container] import`   

docker load与docker import的区别和联系：
联系：docker load用来导入镜像存储文件到本地镜像库，docker import用来导入一个容器快照到本地镜像库。
区别：容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积更大。此外，从容器快照文件导入时可以重新指定标签等元数据信息。
