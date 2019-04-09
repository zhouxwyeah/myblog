---
# 常用定义
title: "Devops安全测试"           # 标题
date: 2019-04-09T10:01:23+08:00    # 创建时间
lastmod: 2019-04-09T10:01:23+08:00 # 最后修改时间
draft: false                       # 是否是草稿？
tags: ["devops"]  # 标签
categories: ["devops"]              # 分类
author: "zhouxwyeah"                  # 作者
description: "Devops安全测试"
# 用户自定义
# 你可以选择 关闭(false) 或者打开(true) 以下选项
comment: true   # 关闭评论
toc: true       # 关闭文章目录
# 你同样可以自定义文章的版权规则
reward: false	 # 关闭打赏
mathjax: true    # 打开 mathjax


---

[AutomatedSecurityTesting](https://slides.com/kiview/securitytesting-greach#/)思路不错，里面的流程和pipeline，以及工具链可以参考，集成到现有的devops流程里面去。

下面针对slide的一些笔记。

#### 渗透测试的问题

* 介入太早  没有生产级别可测试
* 介入太晚，影响进度

#### 介入时机

* 集成安全到devops流程
* 工具化
* 对于复杂案例使用渗透测试
* 培养开发的安全意识

#### Devops

* 传统devops

![image-20190409122622023](../images/image-20190409122622023.png)

* 集成安全的devops

![image-20190409122555646](../images/image-20190409122555646.png)

#### 工具链

##### 代码构建阶段

* 目标

  * 源代码
  * 字节码
  * 依赖包

* 工具

  * 依赖分析

  商业： synk      blackducksoftware

  开源    **DependencyCheck**

  * 静态代码安全扫描

    通用的sonar pde checkstyle findbugs

    商业  veracode   checkmax   mircofocus

    开源 spotbugs(java)  find-sec-bugs(java web, android) 

##### 容器构建阶段

* 容器脆弱性扫描

  * 开源

  clair

  * 商业

  layeredinsight

  docker ee

##### UAT/生产阶段

* 动态扫描
* owsap top 10
* 工具
  * 开源 zaproxy
  * 商业 portswigger   有社区版