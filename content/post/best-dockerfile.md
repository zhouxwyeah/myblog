---
# 常用定义
title: "Dockerfile 最佳实践"           # 标题
date: 2019-02-15T10:01:23+08:00    # 创建时间
lastmod: 2019-02-15T10:01:23+08:00 # 最后修改时间
draft: false                       # 是否是草稿？
tags: ["Docker", "官方文档"]  # 标签
categories: ["tech"]              # 分类
author: "zhouxwyeah"                  # 作者
description: "Dockerfile通用规则以及指令的最佳实践，减少镜像大小和使用误区"
# 用户自定义
# 你可以选择 关闭(false) 或者打开(true) 以下选项
comment: true   # 关闭评论
toc: true       # 关闭文章目录
# 你同样可以自定义文章的版权规则
reward: false	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

Docker 官方有关于Dockerfile最佳实践的一些推荐,包含使用规则，以及指令的正确用法,有兴趣可查看链接 [最佳实践](https://docs.docker.com/v17.09/engine/userguide/eng-image/dockerfile_best-practices/)



* 通用规则
    - 容器应该是短暂的
    - 使用dockerignore文件
    - 使用multi-stage构建
    - 避免安装不需要的包
    - 每个容器应该只关注一个点
    - 减少layer的数量
    - 多利用build cache
    - 
* Dockerfile指令
    - FROM
    - LABEL
    - RUN
    - CMD
    - EXPOSE
    - ENV
    - ADD or COPY 
    - ENTRYPOINT
    - VOLUMN
    - USER
    - WORKDIR
    - ONBUILD
* 官方repo
* 其他资源
