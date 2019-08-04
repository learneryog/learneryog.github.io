---
title: Docker基础（二）
date: 2019-06-02 17:11:03
cover: /img/Random-img/31.jpg
categories: Docker
tags: Docker
---

## 一、Docker数据管理

> &emsp;&emsp;在实际使用Docker的过程中，往往需要对数据进行持久化，或者需要在多个容器之间进行数据共享，这必然涉容器的数据管理操作。
> &emsp;&emsp;容器中的数据管理主要由两种方式：
> &emsp;&emsp;1.数据卷：容器内数据直接映射到本地主机环境；
> &emsp;&emsp;2.数据卷容器：使用特定容器维护数据卷。

### 1、数据卷
数据卷是一个可供容器使用的特殊目录，它将主机操作系统目录直接映射进容器，类似Linux中的mount行为。

数据卷提供了很多有用的特性：
- 数据卷可以在容器之间共享和重用，容器间传递数据将变得高效与方便；
- 对数据卷内数据的修改会立刻生效，无论是容器内操作还是本地操作；
- 对数据卷的更新不会影响镜像，解耦开应用和数据；
- 卷会一直存在，知道没有容器使用，可以安全地卸载它。
#### 1.1. 创建数据卷
> $ docker volume create -d local test
test
$ sudo ls -l /var/lib/docker/volumes
总用量 28
-rw------- 1 root root 32768 6月   2 17:39 metadata.db
drwxr-xr-x 3 root root  4096 6月   2 17:39 test

除了create子命令外，volume还支持inspect（查看详细信息）、ls（列出已有数据卷）、prune（清除无用数据卷）、rm（删除数据卷）等。
#### 1.2. 绑定数据卷
在创建容器时将主机本地的任意路径挂载到容器内作为数据卷，这种形式创建的数据卷称为绑定数据卷。

docker run命令使用-mount选项来使用数据卷。该选项支持三种类型的数据卷：
- volume：普通数据卷，映射到主机/var/lib/docker/volumes路径下；
- bind：绑定数据卷，映射到主机指定路径下；
- tmpfs：临时数据卷 ，只存在于内存中。

> 使用training/webapp镜像创建一个web容器，并创建一个数据卷挂载到容器的/opt/webapp目录：
> $ docker run -d -P --name web --mount type=bind,source=/webapp,destination=/opt/webapp training/webapp python app.py
4c8d75fe28918c96c610200c1fb164dc1e10cbc78b93eef1000e26e3fa328619

注意：本地目录必须是绝对路径，容器内目录可以为相对路径。
### 2、数据卷容器

### 3、利用数据卷容器来迁移数据
