---
layout: post
title: "Java后端工程师学习大纲"
date: 2016-06-27 21:39:34 +0800
comments: true
categories: java
---

之前自己总结过的[Java后端工程师技能树](http://www.rowkey.me/blog/2016/06/17/java-skill-tree/)，其涵盖的技术点比较全面，并非一朝一夕能够全部覆盖到的。对于一些还没有入门或者刚刚入门的Java后端工程师，如果一下子需要学习如此多的知识，想必很多人会望而却步。

本文截取了技能树中的一些关键技能点，并辅以学习资料和书籍推荐，做为Java后端工程师的一个入门或者入职学习计划，基本上涵盖了一个合格的Java后端工程师必备的技能点，是一个相对完整的从基础到高级的修炼过程。当然，这只是一个大纲性指引的东西，也主要针对的是Java后端这个职位，并不会面面俱到，也不会很详细的讲述。毕竟其中每一个知识点深入下去都是可以成书的。另外，像数据结构、计算机网络等计算机科学基础知识，我认为是从事计算机专业的人必备的知识点，因此并不包括在内。如果要一个很全的知识点可以移步[Java后端工程师技能树](http://www.rowkey.me/blog/2016/06/17/java-skill-tree/)。

**本大纲于2016.07.07最新更新**^_^...

<!--more-->

## 一. Git版本管理+Maven工程管理

[微博新兵训练营课程——环境与工具](http://weibo.com/p/1001643874239169320051)

## 二. Java编程

### 书籍

- [《Java核心技术(卷1)》](https://book.douban.com/subject/3146174/)：学习java必备的黄皮书，入门推荐书籍
- [《Java核心技术(卷2)》](https://book.douban.com/subject/3360866/)：黄皮书之高级特性
- [《Java并发编程实战》](https://book.douban.com/subject/10484692/): 对java并发库讲得非常透彻
- [《Effective Java》](https://book.douban.com/subject/3360807/)：Java之父高司令都称赞的一本java进阶书籍
- [《写给大忙人看的Java SE 8》](https://book.douban.com/subject/26274206/):涵盖了java8带来以及java7中被略过的新的java特性，值得一看

### 资料

- Socket编程: <http://ifeve.com/java-socket/>
- NIO: <http://ifeve.com/java-nio-all/>
- 序列化: <http://ifeve.com/java-io-s-objectinputstream-objectoutputstream/>
- RPC框架: <http://dubbo.io>
- 并发编程：<http://ifeve.com/java-concurrency-constructs/>

## 三. 开发框架

- Spring: [跟开涛学Spring3](http://www.open-open.com/doc/view/5407635b943d410c9cfde409c90450b7)
- Spring MVC: [跟开涛学SpringMvc](http://www.cnblogs.com/kaitao/archive/2012/07/16/2593441.html)
- MyBatis: [MyBatis实战教程](http://www.yihaomen.com/article/java/302.htm) [MyBatis学习](http://limingnihao.iteye.com/blog/781671)

对于这些框架或者是一些常用的软件，个人最推崇的还是阅读**官方文档**来学习。当然，看这些资料能让你入门地更加快速一些。

更进一步的，在学会使用之后，去阅读这些框架或软件的源码是必不可少的一步。阅读源码的一种比较好的步骤如下：

- 1) 先阅读架构文档
- 2) 根据架构，将源码文件以模块（或上下层级）分类
- 3) 从最独立（依赖性最小）的模块代码读起
- 4) 阅读该模块功能文档
- 5) 阅读该模块源代码
- 6) 一边阅读一边整理「调用关系表」
- 7) goto 3

## 四. 性能优化与诊断-系统

[《Linux服务器性能调整》](https://book.douban.com/subject/4027746/)

学习内容：

+ Linux概述
+ 性能分析工具
+ 系统调优
+ Linux服务器应用的性能特征
+ 调优案例分析

## 五. 性能优化与诊断-JVM

- [《Java性能权威指南》](https://book.douban.com/subject/26740520/): Java性能调优的权威书籍，几乎涵盖了Java调优的方方面面。
- [《深入理解Java虚拟机(第二版)》](https://book.douban.com/subject/24722612/): 讲解了JVM的内存、GC、字节码、编译器等高级特性和优化实践。

## 六. 消息中间件

### JMS

最为经典，也比较简单的一个消息中间件规范，ActiveMQ是其一个实现。但由于自身的一些局限，不再推荐使用。

+ 大规模分布式消息中间件简介：<http://blog.csdn.net/huyiyang2010/article/details/5969944>
+ JMS Overview: <http://docs.oracle.com/javaee/6/tutorial/doc/bncdr.html>
+ Basic JMS API Concepts: <http://docs.oracle.com/javaee/6/tutorial/doc/bncdx.html>
+ The JMS API Programming Model: <http://docs.oracle.com/javaee/6/tutorial/doc/bnceh.html>
+ Creating Robust JMS Applications:<http://docs.oracle.com/javaee/6/tutorial/doc/bncfu.html>
+ Using the JMS API in Java EE Applications: <http://docs.oracle.com/javaee/6/tutorial/doc/bncgl.html>
+ Further Information about JMS: <http://docs.oracle.com/javaee/6/tutorial/doc/bncgu.html>

### RabbitMQ

RabbitMQ是AMQP(The Advanced Message Queuing Protocol)协议的实现。适用于需要事务管理、对消息丢失很敏感的应用场景。对比kafka来看，RabbitMQ更为强调消息的可靠性、事务等。通过阅读官方文档学习即可：[官方文档](http://www.rabbitmq.com/documentation.html)

### Kafka

基于日志的消息队列，首推当然是官方文档: <http://kafka.apache.org/documentation.html>

- [kafka中文教程](http://www.orchome.com/kafka/index)：比较不错的中文教程

	学习内容：

	+ 开始学习kafka
	+ 入门
	+ 接口
	+ 配置
	+ 设计
	+ 实现
	+ 什么是kafka
	+ 什么场景下使用kafka

- [kafka-study](https://github.com/superhj1987/kafka-study): 笔者在学习kafka时的一些笔记

## 七. OAuth认证技术

### 原理

OAuth是目前最为流行的第三方认证技术，即如何为第三方应用提供基于自己系统帐户体系的认证。目前，微博、微信、QQ、Facebook、Twitter基本上都是通过此协议让第三方应用集成的。简单的介绍可见百度百科简介: [OAuth](http://baike.baidu.com/link?url=Atszf_5BaipVU0_H2Gy8qZ9K0W9WnnmEmRwl6SXkHJyrbB5-GxZ_Kc57hjaCEfF-0wGkcblothOuji0Cabwvu_)。

此外，这里有一篇博文讲的比较详细：[OAuth的机制原理讲解及开发流程](https://www.baidu.com/link?url=dsh9gFpNCLJSQoBq13Pw_nND3XvhBEfuuWQIyDpSDahpKPARnW2b950PgL0ywr8f&wd=&eqid=921a63a50002869300000004577e6e05)。

### 开源实现

- Google oauth core：<http://oauth.net/code/>
- Spring oauth: <http://projects.spring.io/spring-security-oauth/>

## 八. Redis设计与实现

- [Redis命令](http://redisdoc.com/): 使用当然要看这份权威文档，也是平常开发中最常用的参考资料。

- [Redis设计与实现](http://redisbook.com/)：可以通过此文档来学习Redis的原理。当然，自己去看redis的源代码更是不错的选择。

	学习内容：

	+ 常用命令以及数据结构
	+ 内部数据结构
	+ 内存映射数据库结构
	+ redis数据类型
	+ 功能的实现
	+ 内部运作机制

## 九. 数据相关

### 理论基础

+ [MapReduce](http://blog.csdn.net/active1001/archive/2007/07/02/1675920.aspx): 分布式计算的鼻祖，当然谷歌现在推出了新的计算模型。
+ [GFS](http://blog.csdn.net/xuleicsu/archive/2005/11/10/526386.aspx): 分布式存储技术，开源实现为HDFS
+ [Bigtable](http://blog.csdn.net/accesine960/archive/2006/02/09/595628.aspx): 稀疏大型数据库(列数据库)技术，开源实现为HBASE。

作为业界良心的google还有其他许多先进的分布式技术，其论文也非常值得去研读。可以通过此链接获取一些论文的内容：<http://www.chinacloud.cn/show.aspx?id=14382&cid=11>

### 实时计算

+ [《Storm分布式实时计算模式》](https://book.douban.com/subject/26312249/)：虽然twitter推出了新一代的Heron，但Storm仍是目前应用最为广泛的实时计算技术。
+ [Spark streaming: ](https://spark.apache.org/streaming/)：Spark带来了基于批处理的实时流计算技术，对比Storm各有优劣。

### 离线计算

+ [《Hadoop权威指南》](https://book.douban.com/subject/26206050/)：无须多言，Haoop是大数据必须要学习的技术，涵盖了HDFS+HBase+MapReduce。
+ [《Hive编程指南》](https://book.douban.com/subject/25791255/)：Hive降低了MapReduce程序编写的复杂度。
+ [《Spark快速大数据分析》](https://book.douban.com/subject/26616244/)： Spark引进的基于RDD的计算模型大大提高了离线计算的性能，相对于MR来说是更为领先的离线计算技术。

### Lambda架构

大数据领域的经典架构方案，融合了离线和实时计算模型，对外能够提供稳定可靠的数据。对此架构的剖析可见此篇文章：[Linkedln技术高管Jay Kreps：Lambda架构剖析](http://www.csdn.net/article/2014-07-08/2820562-Lambda-Linkedln)

### 机器学习

除了个性化推荐系统之外，CTR预估、广告推荐、预测模型都是机器学习的应用场景。

+ [《推荐系统实践》](https://book.douban.com/subject/10769749/)：推荐系统入门级的读物，强烈推荐初学者学习
+ [《计算广告》](https://book.douban.com/subject/26596778/)：行业大牛的计算广告总结，也是北大的一门课程
+ [《集体智慧》](https://book.douban.com/subject/3288908/)：从集体智慧引出了机器学习相关的技术、算法等
+ [《机器学习》](https://book.douban.com/subject/26708119/)：业界闻名的西瓜书，囊括了基本的机器学习理论知识
+ [《机器学习实战》](https://book.douban.com/subject/24703171/)：以《机器学习》为理论学习材料，结合此本作为代码参考，能够大大提高机器学习的入门速度