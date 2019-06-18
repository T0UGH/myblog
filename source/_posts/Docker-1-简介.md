---
title: '[Docker][1][简介]'
date: 2019-05-22 23:19:36
tags:
    - Docker
categories:
    - Docker
---
## 1 简介

点此进入[play with docker](https://labs.play-with-docker.com/)

### 1.1 容器发展之路

1. 曾经，每个服务器只能运行单一应用。Windows和Linux操作系统都没有相应的技术手段来保证在一台服务器上稳定运行多个应用

2. VM技术是一种允许多应用能够稳定、安全同时运行在一个服务器上的技术

3. 虚拟机缺点是依靠其专用的操作系统，OS会额外消耗资源

4. 容器模型与虚拟机模型的区别是容器的运行不会独占操作系统

5. 容器的优点是启动快且便于迁移

6. Kubernetes是Docker之上的一个平台，现在采用Docker实现其底层容器相关的操作

### 1.2 走进Docker

- Docker是一种运行于Linux和Windows上的软件，用于创建、管理和编排容器

- Docker一词源于英国口语，意为码头工人，即从船上装卸货物的人

- Moby是Github上Docker的开源项目

- 许多Docker内置的组件都是可拔插的，即可以替换为第三方的组件

- OCI(开放容器计划)是一个旨在对容器基础架构中的基础组件进行标准化的机构。目前已经制订了镜像规范和运行时规范，将容器生态团结起来

### 1.3 Docker安装

- win10专业版，进bios开虚拟化，启动Hyper-x，去[这里](https://hub.docker.com/editions/community/docker-ce-desktop-windows)现在Docker for Windows版

- 每个Docker容器都有一个本地存储空间，本地存储是通过存储驱动(Storage Driver)进行管理的

### 1.4 纵观Docker

#### 1.4.1 运维视角

- Docker有两个主要组件:Docker客户端和Docker deamon(服务端或者引擎)

- 镜像
    - 可将镜像理解为一个包含基础操作系统，以及应用程序运行所需的代码和依赖包
    - 实际等价于未运行的容器
    - Docker的每个镜像都有自己唯一的id

- 容器
    - 可以从容器来启动镜像
    - `docker run`可以运行镜像
    - `Ctrl-PQ`组合键可以在退出容器的同时保证容器运行
    - `docker ls`列出系统内部处于运行状态的容器

#### 1.4.2 开发角度

- 容器即应用

- 每个仓库都有一个名为`Dockerfile`的文件。`Dockerfile`是一个纯文本文件，其中描述了如何将应用构建到`Docker`镜像中

- 使用`docker image build`命令可以根据Dockerfile中的指令来创建新的镜像

### 1.5 Docker引擎

- Docker引擎是用来运行和管理容器的核心软件

- Docker引擎可以进行模块化，有很多可交换的组件组成

- runc: 用于创建容器的组件

- containerd: 用于容器生命周期管理

- daemon:之前Docker的服务器，目前其中的越来越多功能被拆解出来


