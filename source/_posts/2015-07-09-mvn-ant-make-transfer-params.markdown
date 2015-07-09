---
layout: post
title: "make、ant、mvn的命令行参数传递"
date: 2015-07-09 11:48:47 +0800
comments: true
categories: 
---

在使用make、ant、mvn的时候，怎样传递命令行参数呢。比如

- 在makefile中读取make后面跟的参数
- 在build.xml中读取ant后面跟的参数
- 在pom.xml中读取mvn后跟的参数

## 1. make

执行命令


	make name=test
在makefile中

	${name}
2. ant

执行命令

	ant -Dname=test
在build.xml中

	${name}
3. mvn

执行命令

	mvn -Dname=test
在build.xml中

	${name}