---
layout:     post
title:      通过DCOS及marathon-lb实现负载均衡与服务发现：一
date:       2016-10-18 18:06:21
summary:    拙译
categories: mesosphere
thumbnail: fa-sign-language
tags:
 - mesosphere
 - DCOS
 - marathon-lb
---

# 通过DCOS及marathon-lb实现负载均衡与服务发现：一


[原文][1]


服务发现和负载均衡系列文章有两部分，本文为第一部分。


Mesosphere Datacenter Operating System（DCOS）为服务发现和负载均衡提供了有用的工具。其中很重要的一个就是Marathon Load Balancer，接下来将围绕他进行展开。


当你启动一个DCOS集群后，所有task可以通过使用Mesos-DNS被发现。通过DNS进行服务发现的话，会有如下的限制：

- DNS不能标示服务端口，除非你用SRV查询；大部分的应用并不能“开箱即用”的使用SRV记录。
- DNS做不到快速failover。
- DNS记录是有TTL（Time to live）的并且Mesos-DNS是通过polling来创建DNS记录的；这回影响记录的实时性。
- DNS记录并不提供任何的服务健康数据。
- 一些应用或者库并不能正确的处理多条A记录；在一些情况下这些查询可能会被缓存并不能像期待的一样被更新。

要解决这些情况，我们为Marathon提供了一个叫做Marathon Load Balancer的工具，简称marathon-lb。

Marathon-lb是基于HAProxy的，是一个快速的代理和负载均衡器。HAProxy可以为基于TCP和HTTP的应用提供代理和负载均衡，并支持如SSL、HTTP压缩、健康检查、Lua脚本等等的功能。Marathon-lb订阅Marathon的事件，可以实时的更新HAProxy的配置。

你可以将marathon-lb配置成不同的拓扑结构。这里关于如何使用marathon-lb提供一些例子：

- 将marathon-lb用作公网负载均衡和服务发现机制。用marathon-lb路由公网的流量。你可以在内网或外网（取决于你的业务）DNS中将外网出口的IP地址配成A记录。


[1]: https://mesosphere.com/blog/2015/12/04/dcos-marathon-lb/