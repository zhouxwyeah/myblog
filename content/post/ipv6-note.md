---
# 常用定义
title: "java ipv6网络用户指南"           # 标题
date: 2019-04-08T10:01:23+08:00    # 创建时间
lastmod: 2019-04-08T10:01:23+08:00 # 最后修改时间
draft: false                       # 是否是草稿？
tags: ["ipv6"]  # 标签
categories: ["tech"]              # 分类
author: "zhouxwyeah"                  # 作者
description: "java 网络IPv6改造"
# 用户自定义
# 你可以选择 关闭(false) 或者打开(true) 以下选项
comment: true   # 关闭评论
toc: true       # 关闭文章目录
# 你同样可以自定义文章的版权规则
reward: false	 # 关闭打赏
mathjax: true    # 打开 mathjax


---

国内工信部的推进下，各大公司应用纷纷支持IPV6，人行也发文要求金融机构应用支持IPV6，本文参考[网络IPV6用户指南](https://docs.oracle.com/javase/8/docs/technotes/guides/net/ipv6_guide/index.html);分析ipv6对于应用改造的影响。

#### ipv4和ipv6 

IPv4长期以来都是互联网上传输数据的Ip协议的行业标准，IPV6作为下一代，很长一段时间我们都需要同时支持IPV4和IPV6，作为应用，我们要支持他们，首先来看看他们有什么不同。

* 第一，最大的不同，就是长度不一样了，IPV4 32位，每8位句号分割，10进制格式书写。IPV6 128位，每16位冒号分割，16进制书写，16位就是4个字符。
* IPV4不能被用作ipv6，但是ipv6能够支持表达ipv4地址，ipv4-mapping地址，如果使用ipv4-mapped，ipv6的前80位为0，后面16为1，32位使用ipv4地址。
* 所以ipv6可以支持ipv6和ipv4-mapped,但是在solaris或者低版本的linux还不支持ipv4-mapped，这个时候只能指定版本了

#### ipv6网络开发

ipv6对于java是透明的，对于socket程序在编程开发上的影响基本没有，原来在ipv4上跑的程序，依然可以不做修改的跑在ipv6模式下。

##### ipv6如何工作在java平台

java网络栈首先会检查操作系统对于ipv6的支持，如果支持，会尝试使用ipv6栈，同时，在双栈系统上，他会建立ipv6 socket。java运行时建立两个socket，分别对应v4和v6的通讯。对于客户端程序，一旦链接建立，ip协议族就确认了，例外一个socket就关闭了。对于服务端，无法确认下一客户端的ip协议族，两个socket都需要同时保持。对于UDP应用，两个socket都需要保持在通讯的生命周期内。

##### 未指定绑定地址

例如 **::** ，对应ipv4的**0.0.0.0** ,有时候也称为anylocal 或者 wildcard ，如果指定 **::** socket在双栈的机器，他能接受ipv4和ipv6的流量;如果指定ipv4（ipv4-mapped），他只能接受ipv4的流量。除非系统属性指定使用ipv4栈，java runtime优先会尝试绑定ipv6 **::**

当绑定**::**,ServerSocket.accept将接受ipv4和ipv6的请求，java api目前没有提供只接受ipv6链接的方法。

尽管如此，一个新的socket属性能够改变这个行为，一个新的ip级别的socket属性，**IPV6_V6ONLY**.这个属性严格约束 AF_INET6只支持ipv6的链接。目前默认该属性是off，而且当前javase的API还不支持该属性，但也不排除会出现在未来的版本中。

##### 环回路地址

**(::1 或者 127.0.0.1)** 环回路包，ipv4和ipv6，都只接受自身的包，而不会处理对方的包

##### ipv4-mapped地址

使用ipv6的方式表示ipv4，如上面表述。

##### 相关属性

java.net.preferIPv4Stack=<true|false> 默认false

java.net.preferIPv6Stack=<true|false> 默认false

##### 通讯场景

在未来大部分时间里，ipv4和ipv6都会共存，平滑过度将是各开发平台需要考虑的事情。对于java来说，一个通用的做法是在双栈的节点上，使用ipv6 socket同时处理ipv4和ipv6请求，在native级别，对待ipv4也是适应ipv4-mapped来模拟ipv6，对应程序来说是屏蔽的

| 节点  | V4     | V4 V6 | V6     |
| ----- | ------ | ----- | ------ |
| V4    | 兼容   | 兼容  | 不兼容 |
| V4 V6 | 兼容   | 兼容  | 兼容   |
| V6    | 不兼容 | 兼容  | 兼容   |

双栈节点，是能够和v4 only,v6 only通讯的，例如host1,host2

* host1 server,host2 client, 如果host2请求host1,他会建立V6 socket，然后查看host1的ip，如果host1只支持v4 协议栈,他将查看name serviced的ipv4条目，host2将使用ipv4-mapped地址请求host1,一个v4的包将由host2发出，host1也以为他通讯的是v4客户端。
* host1 client ,host2 server, host2 是server,首先建立v6 socket，等待链接。host只支持v4，建立v4 socket，他解析域名，获得host2 v4的 IP，使用v4地址链接host2, host2将v4 packet转为ipv4-mapped，应用程序依然处理v6请求一样处理v4链接

##### 获取IP

JAVA获取IP，通过nameservice，可以指定对应的nameservice provider 。

NIS有多个provider,按照顺序配置，根据优先级依次查找

默认使用系统配置的NIS

`sun.net.spi.nameservice.provider.<n>=<default|dns,sun|...>`

指定 DNS  SERVER的IP地址

`sun.net.spi.nameservice.nameservers=<server1_ipaddr,server2_ipaddr ...>`

指定DNS的域名

 `sun.net.spi.nameservice.domain=<domainname>`

#### 应用改造的影响

* 长度，对于数据库长度的字段的变化，很多时候我们会存储ip地址，v6的升级，数据库字段首先被影响
* 网络程序，尽量支持双栈。 里面是否有绑定IP，如果有，需要判断是否支持V6流量的接入。
* 操作系统，需要配置V6的IP
* 应用程序逻辑，是否有根据IPv4的格式去判断ipv6，需要兼容IPv6
* 运维工具，是否有关于ip的配置和限制
* 第三方包，需要验证对于ipv6的兼容性