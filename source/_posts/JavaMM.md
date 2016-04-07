# Java内存管理

## 一. 背景知识

目前国内外著名的几个大型互联网的语言选型如下所示：

1. Google: c/c++ python java js，不得不提的是Google贡献给java社区的guava包质量非常高。
1. Youtube、豆瓣: python
1. fackbook、yahoo、flickr、新浪：**php**(优化过的php) 
1. 网易、阿里、搜狐: java、php、node.js
1. Twitter: ruby->java,之所以如此就在于与jvm相比，Ruby的runtime是非常慢的。并且ruby的应用比起java还是比较小众的。

可见，虽然最近这些年很多言论都号称java已死或者不久即死，但是Java的语言应用占有率一直居高不下。与高性能的c/c++相比，java具有gc机制，并且没有那让人望而生畏的指针，上手门槛相对较低；而与上手成本更低的php、ruby来说，又比这些脚本语言有性能上的优势(这里暂时忽略fb自己开发的php vm)。

对于Java来说，最终是要依靠字节码运行在jvm上的。目前，常见的jvm有以下几种：

- Sun HotSpot
- BEA Jrockit
- IBM J9
- Dalvik(Android)

其中以HotSpot应用最广泛。

## 二. Jvm虚拟机内存简介

## 三. 垃圾收集

### 3.1 理论基础

### 3.2 hotspot垃圾收集器分析

### 3.3 调优经验

