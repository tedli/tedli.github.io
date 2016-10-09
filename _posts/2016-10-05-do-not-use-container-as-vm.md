---
layout:     post
title:      不要把容器当虚拟机用
date:       2016-10-05 19:41:26
summary:    Docker由内核的cgroup和namespace等机制实现进程级隔离，不应该将容器用作虚拟机
categories: Docker
thumbnail: linux
tags:
 - Docker
 - container
 - virtual machine
---


# 不应把容器用作虚拟机

如题，“*不应该*”不是“*不能够*”。自从接触`Docker`，发现有相当一部分人将容器用作虚拟机。比如在容器内开`ssh`服务，开发者可以通过`ssh`登录到容器内，维护容器内业务代码。手段不乏在容器内开`supervisor`等。
`Docker`确实可以这么做，确实可以实现这种效果，并且足够轻量级。这是事实，这里想讨论的不是否认这一事实，而是想说明`Docker`能做的更多，不应该把它用作虚拟机。


## StackOverflow上关于容器与虚拟机的讨论

《[How is Docker different from a normal virtual machine][1]》


## 容器是进程级的

在宿主机`ps`，结果会包含容器内的进程。[这里][2]有`Docker`的原理的一系列文章，及通过代码讲述`Docker`的工作原理。


## 如果用作虚拟机会有问题

出处：《[Containers Are Not VMs][3]》

1. 如何备份一个容器？
2. 用什么策略维护运行中的容器？
3. 业务实例跑在哪台宿主机上？


## 解决问题

出现这3个问题的根本原因是把容器当虚拟机用了。

### 容器可以备份

容器可以`commit`成镜像。但是不应该这么做。容器应该无状态、且为独立的微服务。状态应该根据具体业务需要，保存在数据库、外部数据卷等处。不应该通过`commit`保存镜像，而应该遵从官方的建议镜像只从`Dockerfile`生成。

### 容器可以运维

容器可以通过开各种远程服务，如`ssh`，以便开发者登录到容器内对容器进行修改。前提是容器的`ENTRYPOINT`不能退出，不然容器也会退出。而且对基于同一镜像的多容器副本的某一个实例进行的修改，不会同步到其他实例上，要每一个都登录进去进行相同的修改后，才实现一次“升级”。

### 容器可以定位到

可以到每个`Docker`宿主机上`docker ps`（可加`-a`参数，以便显示停止状态的容器）查看本宿主机上的容器实例。运行状态下的，可以`docker exec`到容器内。


## 带来了新麻烦

如果把容器用作虚拟机，所遇到的问题虽然都可以通过某种方式解决，但不难发现，通过此种方式反而引入了很多不必要的麻烦，进而降低了开发效率。一群顶尖开发者创造`Docker`显然不是为了给本已经非常复杂的软件开发再填新麻烦，一定是为了将软件从业人员从不必要的麻烦中解脱出来。


## [DevOps][4]

`DevOps`的定义不够直观。并且根据开发者接触的业务不同、开发经历不同理解及实现方式也不同。但笼统的来讲他是一个除了开发编码需要人来做之外的自动化的生产过程。常见的有自动化测试、自动化部署、自动化运维、持续集成版本控制等。


## 合理使用Docker

1. 拆分业务为多个独立的微服务；
2. 搭建持续集成、版本控制系统；
3. 每个独立的微服务可以单元测试；
4. 容器有日志收集策略；
5. 有基于容器的发布策略；
6. 有容器间服务发现策略；
7. 有微服务多实例的负载均衡策略；
8. 有`hotfix`迭代策略；
9. 运行独立微服务的容器可以被健康检查；
10. 有基于容器健康检查的集群调度策略；

基于如上手段，`Docker`才正如其字面意思，使不同开发者负责的不同业务足够模块化。由于模块化，开发者便可以专注代码的编写，将版本控制、部署、运维等工序交由自动化系统完成。


[1]: http://stackoverflow.com/questions/16047306/how-is-docker-different-from-a-normal-virtual-machine
[2]: http://coolshell.cn/articles/17010.html
[3]: https://blog.docker.com/2016/03/containers-are-not-vms/
[4]: http://zh.wikipedia.org/zh-cn/DevOps