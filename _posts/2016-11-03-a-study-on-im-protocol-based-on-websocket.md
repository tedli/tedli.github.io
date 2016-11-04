---
layout:     post
title:      对基于WebSocket的即时通讯协议的学习
date:       2016-11-03 16:23:23
summary:    小规模且能承受一定负载的网页即时通讯的学习总结
categories: instant-messaging
thumbnail: book
tags:
 - instant-messaging
 - websocket
 - protobuf
---


# 对基于WebSocket的即时通讯协议的学习

## 要求

1. 服务端有一定负载量；（“一定”取决于业务）
2. 服务端有平滑伸缩能力；
3. 服务端高可靠性；（聊天内容不应该重、漏）
4. 聊天记录只增、查，不删、改，后端数据库应做调优；（传统数据库或分布式系统）
5. 消息通过WebSocket，消息的序列化协议应该：
    1. 序列化后数据小；
    2. 序列化过程耗运算开销小；
    3. 新消息结构能兼容老消息结构；
    4. 可对消息进行密文处理。
    
    
## WebSocket

对WebSocket了解不多。在意用户如果关掉浏览器，服务端是否还会保持着链接。读了[这个资料][1]。浏览器作为客户端会在关闭窗口时关闭与服务端的连接。


## 序列化协议

初步调研，准备在protobuf上进一步学习。


## 未完待续



[1]: http://stackoverflow.com/questions/4812686/closing-websocket-correctly-html5-javascript
