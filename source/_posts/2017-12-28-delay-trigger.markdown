---
layout: post
title: "如何实现延时触发"
date: 2017-12-28 19:29:34 +0800
comments: true
categories: architecture
---

## 问题

用户购买会员需要下订单，然后如果24小时不付款则订单取消，如何实现24小时后关闭订单？

## 方案

1. 每隔一定的时间扫描所有超时的事件

	这是最容易想到的一种方案。此方案的最关键的两点就是如轮训的频率以及如何高效地获取超时订单。
	- 如果要求比较严格，基本不允许延时的误差，那么每隔一秒轮训一次即可。
	- 采用红黑树或者最小堆存储触发任务，按照触发时间戳排序。如此，能够很快地每次获取超时的任务。实践中，一个很简单的方案就是使用Redis的sortedset存储触发任务，这样只需要使用zrangeByScore获取超时的事件，再使用zremrangeByScore即可删除已经触发的任务。

	此种方案的缺点就在于即使频率到达一秒，也可能会有一秒的误差。且轮训的方式在很多情况下并没有可触发的任务时，会浪费资源。
	
1. 阻塞线程等待时间超时

	此方案思路来自于Nginx中定时器的实现。任务的存储和上面的方案类似，采用最小堆或者红黑树即可。然后选择最近要被触发的任务的时间距离作为阻塞调用epoll_wait的超时。阻塞超时后，依次获取最小触发时间的任务，如果超时则执行。
	
	此种方案的最大优点在于不会浪费空的检查周期。
	
1. 采用环形队列

	此方案详细可以见58沈剑的文章[《1分钟实现“延迟消息”功能》](https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651959961&idx=1&sn=afec02c8dc6db9445ce40821b5336736&chksm=bd2d07458a5a8e5314560620c240b1c4cf3bbf801fc0ab524bd5e8aa8b8ef036cf755d7eb0f6&mpshare=1&scene=1&srcid=1229T7uv1GY7TTA8y5TOqXom&key=60adec318085d825898b74f4b744a386fc3de8df1c45a63d5a6de567aadc4e648680340981e891ba3294a7b0e682a704c038cb35ae5f209b7c1418d63900ac85f52c0f0eff90eee514499e971b733e8c&ascene=0&uin=MTk2MDQxMzA4MQ%3D%3D&devicetype=iMac+MacBookAir7%2C2+OSX+OSX+10.12.6+build(16G29)&version=12010110&nettype=WIFI&lang=zh_CN&fontScale=100&pass_ticket=NnbnyJIPGAsCa%2BIK8WpkAFkD0sEcQH6nzaAAkUFx%2BMBl7MeRWqbYSQth0lcjgcLJ)。大体的思路如下：
	
	采用环形队列，3600个slot，每隔1秒扫描一个slot，检查当前slot里面的所有任务，检查其cycleNum是否为0, 为0则触发，否则cycleNum—1。添加定时事件时，根据扫描指针的当前slot的index和事件触发的时间，计算cycleNum和要放入的slot。
	
	此种方案的本质是栅格化与预计算，相比起前两种方案，大大提升了每次获取可触发任务的效率。但同样存在每次查询任务有可能做无用功的问题。