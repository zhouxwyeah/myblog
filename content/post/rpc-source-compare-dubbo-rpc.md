---
# 常用定义
title: "Rpc框架笔记"           # 标题
date: 2019-03-20T10:01:23+08:00    # 创建时间
lastmod: 2019-03-20T10:01:23+08:00 # 最后修改时间
draft: true                       # 是否是草稿？
tags: ["dubbo", "rpc"]  # 标签
categories: ["tech"]              # 分类
author: "zhouxwyeah"                  # 作者
description: "描述dubbo rpc的流程和设计思路"
# 用户自定义
# 你可以选择 关闭(false) 或者打开(true) 以下选项
comment: true   # 关闭评论
toc: true       # 关闭文章目录
# 你同样可以自定义文章的版权规则
reward: false	 # 关闭打赏
mathjax: true    # 打开 mathjax

---

#### 架构原理

* RPC层
* FilterChain层
* Service层

#### 功能特性

* 服务发布订阅
* 服务路由 随机、轮询、权重、粘滞
* 集群容错 Failover、Failback、Failfast
* 服务调用  同步、异步
* 多协议
* 序列化方式
* 统一配置

#### 性能特性

* 高性能
* 低延迟
* scale能力强

#### 可靠性

* 注册中心可靠  HA、Failover
* 消除单点故障   无状态、集群
* 链路可靠性  心跳、重连

#### 服务治理

* 运行时管控    路由、限流、降级、超时控制
* 服务监控       性能、报表、告警
* 生命周期管理     上线、下线
* 快速定位      日志、可视化、链路跟踪
* 服务安全   鉴权、安全防护

#### Netty

##### 可靠性设计

* TCP层 心跳检测 KEEP-ALIVE
* 协议层心跳  HTTP2 Websocket SMPP等
* 应用层心跳   ping-pong ping-ping
* netty检测 读空闲、写空闲、读写空闲
* 断链重连机制 主动关、被动关，提供的closeFuture可以方便的处理断链的情况
* 消息缓存重发  利用netty的future listener机制
* 优雅停机，利用netty api，同时可自行继承扩展

##### 性能设计

* 通讯模型  异步 非阻塞  NioEventLoop 多路复用，单线程多客户端
* 序列化模型 高性能 高扩展
* 协议 高效
* 线程模型  无锁，派发
* 服务端 BOSS线程组 单端口单个线程组 workgroup IO线程 默认 2*CPU，可逐步调整
* 客户端，如果请求多个后台，每次都新建一个NioEventLoopGroup,服务数量一多，每个EventLoopGroup默认数量为 2* CPU，服务一多，线程数就增加。可以采用所有客户端公用同一个NioEventGroup或者hash分组，公用诺干各线程组

#### 序列化

* 高性能
* 易用性 IDL or no IDL
* 跨语言
* 兼容性
* 协议栈扩展   attachment机制，隐式传参，兼容性扩张性，版本号

#### 负载均衡

* 随机  机器配置不对等
* 轮询 慢提供者累积
* 服务调用时延优先 周期性计算平均时延
* 一致性hash
* 本地路由优先 injvm or innative
* 路由规则
  * 条件匹配
  * 脚本路由
* 自定义路由
  * 灰度升级   地址 区域  终端 号段
  * 服务故障、业务高峰导流
  * 业务强相关
  * 接口实现
  * SPI注入
* 配置化路由
  * 本地配置
  * 配置中心
  * 动态下发 

* Failover 转移

* Failback 返回错误

* FailFast 快速报错

#### 异步

* 通知回调，没法控制回调时机
* Future 自行判断，可以组合何时获取，编程较复杂
* Max(T1,T2……Tn)

#### 鉴权

* 接口握手，判断IP列表，交换ID或者保存IP，后续判断合法性

#### 可靠性设计

* 读写消息发生IO异常
* 心跳读写过程中发生了IO异常
* 对方宕机或者重启，网络不通
* 对方发起管理请求
* 心跳
  * 消息头区别
  * 心跳handler 判断消息头
  * 如果是握手成功，则发送心跳，则使用ctx.executor().sheduleAtFixedRate() 才用单线程，定时频率发送
  * 服务端收到心跳请求，则回复
* 重连
  * 异步链接，感知断链事件，如果channel关闭，则执行重连任务，多级，直到成功

#### 协议扩展性

* attachment字段，允许自行扩展