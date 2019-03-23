# 架构设计

先从整体上来看看RocketMQ的架构设计，然后再介绍每个具体的功能模块的设计与实现。

- [1、架构设计图解](#1)
  - [NameServer Cluster](#1.1)
  - [Broker Cluster](#1.2)
  - [Producer Cluster](#1.3)
  - [Consumer Cluster](#1.3)

<a name="1"></a> 
#### 1、架构设计图

从RocketMQ的官网扒拉了一张简易的架构设计图，能够清楚地了解其整体设计。
![arch](https://github.com/wbear1/rocketmq_blog/blob/master/img/arch/arch.png)

<a name="1.1"></a>
#### NameServer Cluster

NameServer Cluster提供了轻量级的服务发现和路由功能。

* Broker管理：Broker会将自身的信息注册到NameServer，并且会定时发送心跳来管理Broker是否存活。
* 路由管理：NameServer维护了broker和messageQueue之间的映射关系

NameServer将所有的信息保存在memory，由多机来保证多可用，另外，即使全部宕机，重启后broker依然会将这些信息上报。上报的信息包括broker节点地址、topic配置、messageQueue对应的broker节点、broker心跳等。NameServer节点之间不进行通信。

<a name="1.2"></a>
#### Broker Cluster

Broker主要负责消息的存储、投递、查询和HA等功能。按功能划分有如下子模块：

* 通信模块：负责client与broker的通信、broker与nameserver的通信
* client管理
* 存储模块：消息的持久化、
* HA: 在部分broker宕掉的情况下，依然能够提供服务
* 索引服务：建立索引，能够快速地查找消息（大部分消息队列无此功能）

![arch1](https://github.com/wbear1/rocketmq_blog/blob/master/img/arch/arch1.png)

<a name="1.3"></a>
#### Producer Cluster

producer用于投递消息到broker，支持分布式部署，意味着可以多个producer节点往同一个topic或messageQueue发送消息

<a name="1.4"></a>
#### Consumer Cluster

consumer用于消息的消费，支持分布式部署，同时支持集群消费（一条消息只会被相同consumeGroup的多节点中的一台消费）和广播消费（一条消息被所有consumer节点消费，不论consumeGroup是否相同）。