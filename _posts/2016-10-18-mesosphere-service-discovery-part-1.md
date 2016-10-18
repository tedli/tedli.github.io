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


[1]: https://mesosphere.com/blog/2015/12/04/dcos-marathon-lb/