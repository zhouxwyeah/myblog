---
# 常用定义
title: "k8s 副本控制器笔记"           # 标题
date: 2019-02-17T10:01:23+08:00    # 创建时间
lastmod: 2019-02-17T10:01:23+08:00 # 最后修改时间
draft: true                       # 是否是草稿？
tags: ["Docker", "k8s"]  # 标签
categories: ["tech"]              # 分类
author: "zhouxwyeah"                  # 作者
description: "k8s in action读书笔记"   # 用户自定义
comment: false   # 关闭评论
toc: false       # 关闭文章目录
reward: false	 # 关闭打赏
mathjax: true    # 打开 mathjax
---



#### 探针

- liveness probe

  k8s可以通过存活探针来检查容器是否还在运行，有三种探测机制：

  - HTTP GET 针对容器的IP下执行HTTP GET请求，返回码2XX或者3XX表示成功，其他表示失败
  - TCP SOCKET尝试与容器指定端口建立连接
  - EXEC探针执行命令，返回码为0表示成功，非都认为是失败

  #####   HTTP存活探针

  

  