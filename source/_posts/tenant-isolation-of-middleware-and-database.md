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

SaaS应用基本都采用了多租户的架构，以支撑更多的业务。提到多租户，那么一定会提到中间件、数据库等资源如何进行多租户的共享与隔离。
<!-- More -->

以数据库为例，大部分数据隔离方式遵从三种模式：
1. 完全隔离，每个租户独立数据库 
2. 部分共享，租户共享一个数据库，以schema或者table区分
3. 完全共享，租户共享相同的数据库表，以tenant_id列区分

这里说一种可编排的方式，以资源为粒度的隔离，租户可以独享资源，一些租户也可以共享资源。
![Multiple tenant resource config](http://media.kosho.tech/blog/20200206/multiple-tenant-res.png)
租户A：MySQL1(schema=tenant1)，RabbitMQ1，Kafka1，Redis1
租户B：MySQL2(schema=tenant2)，RabbitMQ2，Kafka1，Redis2
租户B：MySQL2(schema=tenant3)，RabbitMQ2，Kafka1，Redis3