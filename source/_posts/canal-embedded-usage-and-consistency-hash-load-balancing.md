---
title: Canal的嵌入式使用与负载均衡改造
date: 2019-10-08 23:21:17
tags:
- Canal
- 负载均衡
categories:
- Canal
- 负载均衡
---

[Canal](https://github.com/alibaba/canal)是阿里巴巴的开源项目，基于MySQL Binlog解析，提供增量数据订阅和消费，是一个广泛使用的数据同步组件。我们的业务也大量使用了Canal，用于ES同步，业务消息通知等场景，分享一下自己的实践。
<!-- More -->

## 直接部署Canal的方式遇到的问题
开始我们按照通常做法，独立部署canal，使用client进行交互，这个过程遇到了一些问题。

### 运维，配置管理
项目环境比较多，每套环境多个租户（独立数据库），时常会新增租户和变更数据库。运维能力比较弱，经常配置不好Canal。我们的数据库配置是作为元数据管理的，就想把Canal集成到系统里，作为一个微服务运行。自动新增和启停instance。

### 缺少负载均衡机制
Canal自身的HA机制基于zk节点抢占实现，没有实现负载均衡。同一套集群运行的instance多了，负载不均的问题就会比较明显，instance很容易都运行在一个节点，造成OOM等问题。
如果是直接部署的方式，server端可以手动配置分割为多个小集群，client端不行。既然已经用了嵌入式启动的方式，就只能自行实现负载均衡了。

### 性能
使用CanalConnector需要多一次网络传输以及序列化，数据量比较大的场景会有性能上的问题，Embedded API就可以绕过这一步，带来比较大的性能提升。

## 扩展后的组件设计
![Canal service](http://pypc1ne42.bkt.clouddn.com/blog/20191013/canal-service.png)

- Channel：在Instance上一层的数据同步通道，管理instance和consumer（poller）
- ChannelPipeline：数据同步流水线
- ChannelHandlers：数据处理handler链
- CanalInstanceCoordinator：负载均衡协调器，依赖zk和ConsistentHashingLoadBalancer进行负载均衡
- ConsistentHashingLoadBalancer：一致性Hash负载均衡
- SubscribeInfoLoad：加载数据订阅信息
- CanalController：同步服务控制器

## Canal的嵌入式使用

### instance生成扩展
Canal默认的方式是扩展spring文件的方式来启动，用SpringCanalInstanceGenerator生成instance，对于每个instance配置文件，都初始化一个spring上下文来生成instance。
官方也有提到他们内部一般是用manager的方式接入，我们可以使用ManagerCanalInstanceGenerator，实现自己的CanalConfigClient加载数据库配置，或者实现自己的CanalInstanceGenerator扩展Canal实例生成机制。
同样，对于instance，可以直接使用CanalInstanceWithManager，或者自行继承AbstractCanalInstance拼接instance，可以加载自己的组件进去，比如说监控。

### Embedded API使用
这时候就可以直接用CanalServerWithEmbedded.instance()获取嵌入式服务对象，来进行订阅消费操作了，与CanalConnector一样。

### Embedded Client
启动server以后，也要同步启动自己的client，我选择复用了ServerRunningMonitor，封装一个Channel对象（同步管道）来管理instance和client，在ServerRunningMonitor#start的时候启动整个channel。

## 基于一致性Hash的负载均衡

### 一致性Hash
网上关于一致性Hash的文章很多，不再赘述它的概念。选用一致性hash作为负载均衡/HA的原因有这么几个：
1. 服务器不变动的情况下，同一个instance会负载分配到一个节点
2. 有节点上下线的情况下，可以让实例的变动尽可能的小

注意使用带有虚拟节点的一致性hash，让instance分布更均衡些。

直接上图，有Canal1，Canal2，Canal3三个Canal节点（每个节点有三个虚拟节点），10个instance按照一致性hash的方式负载到这三个节点上：
![Canal consistent hashing](http://pypc1ne42.bkt.clouddn.com/blog/20191013/canal-consistent-hashing2.png)

负载均衡状态：
- Canal1：instance7，instance8，instance9
- Canal2：instance2，instance3，instance6，instance10
- Canal3：instance1，instance4，instance5

如果2节点下线，负载均衡状态变为：
- Canal1：instance3，instance7，instance8，instance9，instance10
- Canal3：instance1，instance2，instance4，instance5，instance6

### 实现思路
1. Canal节点映射为n个虚拟节点，按照ip-port-seq进行hash，构建hash环，Java可以使用ConcurrentSkipList实现
2. Instance按照name进行hash，映射到hash环上，分配给顺时针查找到的第一个节点
3. 监听zookeeper上Canal的cluster path，当有节点上下线，进行重分配