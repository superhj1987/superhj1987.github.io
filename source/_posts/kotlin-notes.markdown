---
layout: post
title: "Kotlin语法简明指南"
date: 2018-04-10 19:29:34 +0800
comments: true
categories: java kotlin
---

Kotlin是Intellij IDEA的创造团队JetBrains发明的新一代JVM语言。虽然JVM上一次又一次出现新的语言叫嚣着取代Java，但时至今日，Java也开始吸纳其他语言的各种优势，其生命力依旧强盛，生态也越发强大。那么Kotlin的出现是又一次重蹈覆辙还是有其突破性的特性？本文主要对其语法作了简要概括。

<!--more-->
 
1. 没有类型的Java

	虽然Kotlin是静态语言，但其引入的安全类型推断让其无须声明类型。使用val/var即可，其中val定义只读变量，var定义可变变量。
	
	```
	var str1 : String = "a" //有初始值，可以省略类型
	val str2 : String //无初始值，不能省略类型
	str2 = "b"
	var str = "i can change"
	val immutableStr = "i cannot change"
	```


最后，可以从各种资料中总结出Kotlin有以下几个特点：

1. 无缝与Java互操作

	可以直接使用Java库能够大大降低切换语言的成本。因此，如果想选择Kotlin，可以先试着用Kotlin写单元测试，之后等熟悉了此语言，可以进行混写。最后则完全切换到Kotlin。
	
1. 天然对协程的支持

1. 能够编译成原生二进制程序

1. 天然支持函数式编程和面向对象编程

1. 主流三方框架的天然支持 	
	
