---
date: 2024-08-08:36:36+08:00
slug: 'ea'
tags: ["EA"]
---


## 数据访问之 Sharing Rules ｜ 以数据的方式来实现数据访问安全 （修改中...）

### 传统的访问管理设计

### saleforce 如何看待和设计安全的理念

一般的实践

数据库视图 view 

oracl 的表权限


一个强大的数据库，可以省去上层的很多工作

MT_DATA 多租户的自定义数据对象的的存储 https://architect.salesforce.com/fundamentals/platform-multitenant-architecture
还有自定义的 flex 的索引实现

[什么是 Salesforce 数据库](https://noltic.com/stories/what-is-salesforce-database) 讲了利用 oracle 的多种等安全特性

其中 https://www.reco.ai/hub/sharing-rules-in-salesforce 提到性能的问题
规则的性能：
Sharing Rules in Salesforce are available in Professional Edition, Enterprise Edition, Performance/Unlimited Edition, and Developer Edition.
Salesforce 中的共享规则在 Professional Edition、Enterprise Edition、Performance/Unlimited Edition 和 Developer Edition 中可用。
The default limit for sharing rules per object is 300. You can only create up to 300 sharing rules per object. A support case can increase this limit to a maximum of 500.
每个对象的共享规则的默认限制为 300。每个对象最多只能创建 300 个共享规则。支持案例可以将此限制增加到最多 500 个。
The default limit for Criteria-Based sharing rules is 50.
基于标准的共享规则的默认限制为 50。
Creating a Sharing Rule in Salesforce is a straightforward process that can make data access easier.
在 Salesforce 中创建共享规则是一个简单的过程，可以更轻松地访问数据。

数据变更的访问维护：
https://architect.salesforce.com/fundamentals/architecture-basics

实现的戏法 ：
https://www.slideshare.net/slideshow/understanding-the-salesforce-architecture-how-we-do-the-magic-we-do/53849712