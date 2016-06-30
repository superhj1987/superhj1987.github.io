---
layout: post
title: "Java后端工程师学习大纲"
date: 2016-06-27 21:39:34 +0800
comments: true
categories: java
---

之前自己总结过的[Java后端工程师技能树](http://www.rowkey.me/blog/2016/06/17/java-skill-tree/)，其涵盖的技术点比较全面，并非一朝一夕能够全部覆盖到的。对于一些还没有入门或者刚刚入门的Java后端工程师，如果一下子需要学习如此多的知识，想必很多人会望而却步。

本文截取了技能树中的一些关键技能点，并辅以学习资料和书籍推荐，做为Java后端工程师的一个入门或者入职学习计划，是一个相对完整的从基础到高级的修炼过程。基本上涵盖了一个合格的Java后端工程师必备的技能点。

***本大纲会持续更新***

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

- [《Java性能优化权威指南》](https://book.douban.com/subject/25828043/)

	学习内容：

	+ JVM概述
	+ JVM性能监控
	+ JVM性能剖析与工具
	+ JVM参数与调优步骤
	+ JVM调优案例分析

- [《深入理解Java虚拟机(第二版)》](https://book.douban.com/subject/24722612/)

## 六. 消息中间件

### JMS

+ 大规模分布式消息中间件简介：<http://blog.csdn.net/huyiyang2010/article/details/5969944>
+ JMS Overview: <http://docs.oracle.com/javaee/6/tutorial/doc/bncdr.html>
+ Basic JMS API Concepts: <http://docs.oracle.com/javaee/6/tutorial/doc/bncdx.html>
+ The JMS API Programming Model: <http://docs.oracle.com/javaee/6/tutorial/doc/bnceh.html>
+ Creating Robust JMS Applications:<http://docs.oracle.com/javaee/6/tutorial/doc/bncfu.html>
+ Using the JMS API in Java EE Applications: <http://docs.oracle.com/javaee/6/tutorial/doc/bncgl.html>
+ Further Information about JMS: <http://docs.oracle.com/javaee/6/tutorial/doc/bncgu.html>

### Kafka

<http://www.orchome.com/kafka/index>

学习内容：

+ 开始学习kafka
+ 入门
+ 接口
+ 配置
+ 设计
+ 实现
+ 什么是kafka
+ 什么场景下使用kafka

### RabbitMQ

[官方文档](http://www.rabbitmq.com/documentation.html)

## 七. Redis设计与实现

<http://redisbook.com/>

学习内容：

+ 常用命令以及数据结构
+ 内部数据结构
+ 内存映射数据库结构
+ redis数据类型
+ 功能的实现
+ 内部运作机制

## 八. 数据相关

### 理论基础

+ MapReduce：http://blog.csdn.net/active1001/archive/2007/07/02/1675920.aspx
+ GFS：http://blog.csdn.net/xuleicsu/archive/2005/11/10/526386.aspx
+ Bigtable：http://blog.csdn.net/accesine960/archive/2006/02/09/595628.aspx

### 实时计算

+ Storm:[《Storm分布式实时计算模式》](https://book.douban.com/subject/26312249/)
+ Spark streaming: [官网文档](https://spark.apache.org/streaming/)

### 离线计算

+ Hadoop:[《Hadoop权威指南》](https://book.douban.com/subject/26206050/)
+ Hive:[《Hive编程指南》](https://book.douban.com/subject/25791255/)
+ Spark:[《Spark快速大数据分析》](https://book.douban.com/subject/26616244/)

### Lambda架构

[Linkedln技术高管Jay Kreps：Lambda架构剖析](http://www.csdn.net/article/2014-07-08/2820562-Lambda-Linkedln)

### 机器学习：

- [《推荐系统实践》](https://book.douban.com/subject/10769749/)
- [《计算广告》](https://book.douban.com/subject/26596778/) 
- [《集体智慧》](https://book.douban.com/subject/3288908/) 
- [《机器学习》](https://book.douban.com/subject/26708119/)