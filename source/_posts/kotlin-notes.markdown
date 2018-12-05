---
layout: post
title: "Kotlin语法简明指南"
date: 2018-04-10 19:29:34 +0800
comments: true
categories: java kotlin
---

Kotlin是Intellij IDEA的创造团队JetBrains发明的新一代JVM语言。虽然JVM上一次又一次出现新的语言叫嚣着取代Java，但时至今日，Java也开始吸纳其他语言的各种优势，其生命力依旧强盛，生态也越发强大。那么Kotlin的出现是又一次重蹈覆辙还是有其突破性的特性？本文主要对其语法作了简要概括。

<!--more-->

1. 包的定义

	与Java类似，但包的定义与目录结构无需匹配，源代码可以在文件系统任意位置。

	与Java有一点不同，导入包的时候，可以使用import as实现重命名来解决名字冲突的问题。如：
	
	```
	import me.rowkey.MainClass as aClass // aClass 代表“me.rowkey.MainClass”
	```
	
1. 没有类型的Java

	虽然Kotlin是静态语言，但其引入的安全类型推断让其无须声明类型。使用val/var即可，其中val定义只读变量，var定义可变变量。
	
	```
	var str1 : String = "a" //有初始值，可以省略类型
	val str2 : String //无初始值，不能省略类型
	str2 = "b"
	var str = "i can change"
	val immutableStr = "i cannot change"
	```
	
2. 不需要的public

	Kotlin中默认的可见性修饰符是public，所以public修饰符不需要写。其他修饰符如下：
	
	- private：只在类内部/声明文件内部可见。
	- protected：private+子类中可见。
	- internal: 同一模块（编译在一起的一套Kotlin文件）可见。
	
1. 函数定义

	用fun关键字声明函数
	
	```
	fun main(args: Array<String>) {
     ...
	}
	```
	
	其中，函数参数使用 Pascal 表示法定义，即 name: type。参数用逗号隔开。每个参数必须有显式类型。
	
	Kotlin中的函数和Java中的方法是一致的，但与Java不同的是，Kotlin中的函数可以属于任何类，文件当中直接定义则作为“包级函数”，和类的使用方式一致。
		
1. 默认参数值

	函数的参数可以指定默认值。
	
	```
	fun getList(list: Array<String>, offset: Int = 0, size: Int = list.size) { …… }
	```

	不指定第2个参数调用方法时，offset参数取默认值0, size参数默认取第一个参数的size。
	
1. 不需要的语句结束符

	Kotlin中没有语句结束符，当然为了与java保持一致性，也可以使用;号作为语句结束符。
	
1. 字符串连接符
	
	跟java一样，如果你需要把一个字符串写在多行里，可以使用+号连接字符串。代码可以这样写：
   
   ```
   val str = "hello" + "world" + "!!!";
   ```
   
	Kotlin中的写法也可以这样：
	
	```
	val str = """hello
	world
	!!!
	"""
   ```        
          
	三个”号之间不在需要+号进行连接，不过字符串中的格式符都会被保留，包括回车和tab。
	
1. 一切皆对象

	Kotlin中一切皆对象。即使赋值为基本数据类型，也会自动转换为对应的类。

1. 循环

	

最后，可以从各种资料中总结出Kotlin有以下几个特点：

1. 无缝与Java互操作

	可以直接使用Java库能够大大降低切换语言的成本。因此，如果想选择Kotlin，可以先试着用Kotlin写单元测试，之后等熟悉了此语言，可以进行混写。最后则完全切换到Kotlin。
	
1. 天然对协程的支持

1. 能够编译成原生二进制程序

1. 天然支持函数式编程和面向对象编程

1. 主流三方框架的天然支持