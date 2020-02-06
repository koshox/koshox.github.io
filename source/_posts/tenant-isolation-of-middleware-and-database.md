---
title: 中间件、数据库的多租户设计
date: 2020-02-06 01:59:46
tags:
- 中间件
- 数据库
- 多租户
categories:
- 中间件
- 数据库
- 多租户
---

这段时间公司新项目，忙的好久没写文章了，期间做了一些中间件多租户共享与隔离设计。刚好被新冠病毒摁在家里有点时间，分享一下这方面的实践。
SaaS应用基本都采用了多租户的架构，以支撑更多的业务。提到多租户，那么一定会提到中间件、数据库等资源如何进行多租户的共享与隔离。
<!-- More -->

## 多租户隔离模式

以数据库为例，大部分数据隔离方式遵从三种模式：
1. 完全隔离，每个租户独立数据库 
2. 部分共享，租户共享一个数据库，以schema或者table区分
3. 完全共享，租户共享相同的数据库表，以tenant_id列区分

这里说一种基于元数据，可编排的方式。以资源为粒度进行隔离，租户可以独享资源，一些租户也可以共享资源。
![Multiple tenant resource config](http://media.kosho.tech/blog/20200205/multiple-tenant-res.png)
- 租户1：MySQL1(schema=tenant1)，RabbitMQ1，Kafka1，Redis1(index=0)
- 租户2：MySQL2(schema=tenant2)，RabbitMQ2，Kafka1，Redis2(index=0)
- 租户3：MySQL2(schema=tenant3)，RabbitMQ2，Kafka1，Redis3(index=1)

把所有的资源配置作为元数据，存储到配置中心，在系统运行时动态的创建和路由DataSource，RabbitMQ Connection，Redis Pool，路由key为tenantId。

## 资源隔离

### 数据库
基本思想参考spring的AbstractRoutingDataSource，在运行时根据线程上下文中的tenantID动态路由数据源，变更自动刷新。可以按照这种模式构建读写分离，分库分表的数据源，多租户分库分表后面再写篇文章详细说。
![Tenant Datasource](http://media.kosho.tech/blog/20200206/tenant-datasource.png)

### Redis
类似于数据库，只不过把DataSource换成Spring的RedisConnectionFactory，喜欢直接使用Jedis也可以自己封装JedisPool。

### 消息中间件
RabbitMQ这种连接有状态（注册消费者）的资源，比起数据库，Redis更复杂一些，有这么几个原因：
1. 租户隔离变更以后，需要关闭/恢复租户消费者注册。不能直接使用Spring amqp，只能自行封装一套多租户RabbitMQ客户端。我在公司写了一套类Jms Api的RabbitMQ多租户客户端，大概10000行代码，包括各种监控。
2. queue名称添加租户id进行隔离，如order.update -> tenant1.order.update, tenant2.order.update。
3. 数据库资源只需要做到数据源级别复用，对于RabbitMQ来说，Channel也是很宝贵的资源，频繁创建会消耗大量性能，所以Channel也要进行池化复用。

![Tenant RabbitMQ](http://media.kosho.tech/blog/20200206/tenant-rabbitmq.png)

![Tenant RabbitMQ](http://media.kosho.tech/blog/20200206/tenant-rabbitmq-conn.png)