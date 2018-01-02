---
title: "虚拟路由冗余协议（VRRP）详解"
date: 2017-12-30T16:08:31+08:00
categories:
    - 高可用
    - 网络协议
tags: ["VRRP"]
draft: false
---
虚拟路由冗余协议（Virtual Router Redundancy Protocol，简称VRRP）用来消除静态默认路由环境下的单点故障。VRRP阐述了局域网中动态分配职责的选举协议。

## 一些基本定义
### VRRP路由（VRRP Router）
运行VRRP协议的路由设备，它可以参与组成一个或者多个虚拟路由。

### 虚拟路由（Virtual Router）
一个受VRRP协议管理的抽象对象，作为一个共享局域网中的默认路由。它由一个虚拟路由ID（Virtual Router Identifier）和一个或多个关联的IP地址组成。一个VRRP路由可以备份一个或多个虚拟路由。

### IP地址拥有者（IP Address Owner）
将虚拟路由的IP地址作为真实接口地址的VRRP路由。这个设备可以响应发往虚拟路由IP地址的各类协议的数具包如ICMP pings、TCP连接等等。

### 主IP地址（Primary IP Address）
从多个真实接口地址中选择出来的IP地址。一个可选的算法是总是选择第一个地址。VRRP通告报文（VRRP advertisements）总是使用主IP地址作为数具包的源地址。

### Master路由（Virtual Router Master）
承担转发发送给虚拟路由IP地址的数具包和应答对虚拟路由IP地址的ARP请求职责的VRRP路由。如果存在IP地址拥有者，则它总会成为Master。

### Backup路由（Virtual Router Backup）
当前Master路由失效时，可以继续承担转发职责的VRRP路由，可以存在多个。

### 虚拟路由IP地址（IP address(es) associated with the virtual router）
虚拟路由在局域网中关联的IP地址，一个虚拟路由可以具有多个IP地址。

### 虚拟路由MAC地址


## 主要功能
### IP地址备份
IP地址备份是VRRP协议的主要功能。提供Master路由选举的同时还附带以下功能，协议力求：  
- 最小化[黑洞](https://en.wikipedia.org/wiki/Black_hole_(networking))时间  
- 最小化稳定状态的带宽占用和处理复杂度  
- 可用于各种支持IP流量的多路访问局域网技术  
- 提供多个虚拟路由的选举以实现负载均衡  
- 单个网段内多个逻辑IP子网的支持  

### 偏好路径指示
简单的Master选举模式就是把每个路由器作为Master对等地处理，竞争出胜利的一个。 但是，在一组冗余路由器中可能存在许多不同的偏好（或偏好范围）。 例如，该偏好可以基于访问链路的成本或速度，路由器性能或可靠性或其他策略考虑。 该协议应该允许以直观的方式表达该相对路径偏好，并且保证主集中到当前可用的最优路由器。

### 最小化不必要的服务中断
一旦执行了Master选举，Master路由器和Backup路由器之间的任何不必要的转换都可能导致服务中断。 在Master选举之后，协议应该确保只要Master继续正常工作，任何优先级等于或者低于它的备份路由器都不会触发状态转换。

### 扩展的局域网之上的高效操作
在多路访问局域网上发送IP数据包需要从IP地址映射到MAC地址。 在使用[自学习网桥（透明网桥）](https://en.wikipedia.org/wiki/Bridging_(networking))的扩展局域网中使用虚拟路由器MAC地址会对发送到虚拟路由器的数具包的带宽开销具有显著的影响。 如果虚拟路由器MAC地址从不被用作链路层数据帧中的源地址，那么该[station](https://en.wikipedia.org/wiki/Station_(networking))位置永远不会被学习，导致发送到虚拟路由器的所有数具包泛滥（flooding）。 为了提高这种环境下的效率，协议应该：  
- 使用虚拟路由器MAC地址作为Master发送的数据包中的源地址，以触发station学习  
- 转换到Master状态后立即触发一个消息以更新station学习缓存  
- 触发Master的周期性消息来维护station学习缓存  

## VRRP概要
VRRP阐述了虚拟路由的选举协议。协议所有的消息使用IP多播数据报来发送，因此协议可以作用于各种支持IP多播的多路局域网技术。每个虚拟路器由都被分配一个IEEE 802标准的48位MAC地址。虚拟路由的MAC地址被用于所有Master路由发送的周期性的VRRP消息的源地址，用于在扩展的局域网中开启网桥学习功能。  
一个虚拟路由被定义成一个虚拟路由ID（VRID）和一组IP地址。一个VRRP路由可能关联一个使用真实接口地址的虚拟路由，并且也可能被额配置的虚拟路由中被设定为Backup路由。VRID与地址之间的映射对于所有VRIP路由器来说必须是正交的。然而，并不妨碍重用一个VRID在不同的网段中与不同的地址映射。每个虚拟路由的作用域仅仅限制在单独的局域网中。  
为了最小化占用网络流量，每个虚拟路由中只有Master路由发送周期性的的VRRP公告消息。Backup路由不会试图去抢占Master除非它有更高的优先级。这条规则消除了服务中断除非有更优的路径可用。也有可能出于管理的需要禁止所有抢占行为。唯一的例外是IP拥有者总是会成为Master路由。如果Master路由变得不可用则短暂的延时后最高优先级的Backup路由会切换为Master路由，以提供最小化服务中断的虚拟路由职责的切换控制。  
VRRP协议设计提供迅速的从Backup到Master的最小化服务中断的切换方法，并包含减少协议为了保证Master在典型场景下控制切换的复杂度的优化。这个优化导致了最小化的运行时状态需求、最小激活的协议状态以及单独的消息类型和发送者。典型的运营场景是定义两个冗余路由并且／或者对于每个路由区分偏好路径。不遵循这些假设的（例如超过两个冗余路由路径偏好完全相同）的副作用是选举Master时会有短暂的周期重复转发数据包。然而这些典型场景的假设似乎可以覆盖绝大多数的部署方式，低频的Master丢失、预期的Master选举间隔非常小（远小于1秒）。因此当遭到极小概率的短暂网络破坏的时候VRRP的优化在协议设计中表现出很有效的简化。

## 配置样例
### 简单的VRRP路由的网络
```plain
            +-----------+      +-----------+
            |   Rtr1    |      |   Rtr2    |
            |(MR VRID=1)|      |(BR VRID=1)|
            |           |      |           |
    VRID=1  +-----------+      +-----------+
    IP A ---------->*            *<--------- IP B
                    |            |
                    |            |
  ------------------|------------|-----|--------|--------|--------|--
                                       ^        ^        ^        ^
                                       |        |        |        |
                                     (IP A)   (IP A)   (IP A)   (IP A)
                                       |        |        |        |
                                    +--|--+  +--|--+  +--|--+  +--|--+
                                    |  H1 |  |  H2 |  |  H3 |  |  H4 |
                                    +-----+  +-----+  +--|--+  +--|--+
     Legend:
              ---|---|---|--  =  Ethernet, Token Ring, or FDDI
                           H  =  Host computer
                          MR  =  Master Router
                          BR  =  Backup Router
                           *  =  IP Address
                        (IP)  =  default router for hosts             
```
### 同时配置多个虚拟路由实现VRRP路由负载均衡
```plain
            +-----------+      +-----------+
            |   Rtr1    |      |   Rtr2    |
            |(MR VRID=1)|      |(BR VRID=1)|
            |(BR VRID=2)|      |(MR VRID=2)|
    VRID=1  +-----------+      +-----------+  VRID=2
    IP A ---------->*            *<---------- IP B
                    |            |
                    |            |
  ------------------|------------|-----|--------|--------|--------|--
                                       ^        ^        ^        ^
                                       |        |        |        |
                                     (IP A)   (IP A)   (IP B)   (IP B)
                                       |        |        |        |
                                    +--|--+  +--|--+  +--|--+  +--|--+
                                    |  H1 |  |  H2 |  |  H3 |  |  H4 |
                                    +-----+  +-----+  +--|--+  +--|--+
     Legend:
              ---|---|---|--  =  Ethernet, Token Ring, or FDDI
                           H  =  Host computer
                          MR  =  Master Router
                          BR  =  Backup Router
                           *  =  IP Address
                        (IP)  =  default router for hosts             
```

## 协议数据包格式
```plain
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-+
   |Version| Type  | Virtual Rtr ID|   Priority    | Count IP Addrs|
   +-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-+
   |   Auth Type   |   Adver Int   |          Checksum             |
   +-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-+
   |                         IP Address (1)                        |
   +-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-+
   |                            .                                  |
   |                            .                                  |
   |                            .                                  |
   +-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-+
   |                         IP Address (n)                        |
   +-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-+
   |                     Authentication Data (1)                   |
   +-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-+
   |                     Authentication Data (2)                   |
   +-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-+
```
### IP协议字段描述

#### Source Address
被发送的数据包的源地址，固定为主IP地址。

#### Destination Address
IANA分配给VRRP协议的IP多播地址：224.0.0.18。这是一个本地链路层的多播地址。路由器不能忽略TTL转发这个这个目的地址的数据报。

#### TTL
TTL必须设为255。如果VRRP路由器收到TTL不等于255的数据包则必须丢弃。

#### Protocol
IANA分配给VRRP的IP协议号：112（十进制）。

### VRRP协议字段描述
#### Version


#### Type


#### Virtual Rtr ID (VRID)


#### Priority


#### Count IP Addrs


#### Authentication Type


#### Advertisement Interval (Adver Int)


#### Checksum


#### IP Address(es)


#### Authentication Data

## 协议状态机
### 每个虚拟路由的参数
#### VRID

#### Priority

#### IP_Addresses

#### Advertisement_Interval

#### Skew_Time

#### Master_Down_Interval

#### Preempt_Mode

#### Authentication_Type

#### Authentication_Data

### 计时器
#### Master_Down_Timer

#### Adver_Timer

### VRRP状态切换图表
```plain
                      +---------------+
           +--------->|               |<-------------+
           |          |  Initialize   |              |
           |   +------|               |----------+   |
           |   |      +---------------+          |   |
           |   |                                 |   |
           |   V                                 V   |
   +---------------+                       +---------------+
   |               |---------------------->|               |
   |    Master     |                       |    Backup     |
   |               |<----------------------|               |
   +---------------+                       +---------------+
```
![vrrp-state](/vrrp-protocol/vrrp-state.png)  
[点击看大图](/vrrp-protocol/vrrp-state.png)

## TODO
