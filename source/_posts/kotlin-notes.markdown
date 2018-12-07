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
	
1. 字符串模板

	Kotlin提供了$符来做字符串内的变量替换，并且可以做一些字符串操作。如下：
	
	```
	var name = "hj"
	var strTemplate = "My name is $name"//My name is hj
	
	strTemplate = "My name is ${name.replace("j","a")}"// My name is ha
	```
	
1. 一切皆对象

	Kotlin中一切皆对象。即使赋值为基本数据类型，也会自动转换为对应的类。

1. 循环

    Kotlin的while循环和Java没什么不同, 在for循环引入了区间的概念。
    
    ```
    for(i in 1..10){
    	println(i)
    }
    
    for(i in 1..10 step 2){
    	println(i)
    }
    
    for(i in 10 downTo 1 step 1){
    	println(i)
    }
    
    for (i in 1 until 10) {
    	// i in [1, 10) 排除了 10
    	println(i)
	}
    
	for(c in 'A'..'Z'){
    	println(c)
	}
   ```
    
    需要注意的是在Kotlin中不再支持Java的for循环形式：
    
    ```
    for(int i =0;i < 10;i++){
    	...
    }
    ```
1. 操作符重载

	Kotlin提供了操作符重载的支持。对于常用的”+“、"-"等操作符，创建带有operator且名称符合要求的方法，即可实现。如：
	
	```
	data class Point(val x: Int, val y: Int)

	operator fun Point.unaryMinus() = Point(-x, -y)

	val point = Point(10, 20)

	fun main() {
   		println(-point)  // 输出“Point(x=-10, y=-20)”
	}
	```
	
	上面即完成了对-的重载。
    
1.  集合

	Kotlin把集合分为可变集合和不可变集合。其创建需要通过标准库的方法：listOf()、 mutableListOf()、 setOf()、 mutableSetOf()、hashMapOf()、mutableHashMapOf()
	
	```
	val list = listOf("1","2","3",""4)
   val set = setOf("1","2")
   val map = hashMapOf("name" to "hj","sex" to "male")
	```
	
	这些集合类实现了操作符重载，如下：
	
	```
	val list1 = list - listOf("1","2")
	val list2 = list + "2"
	println(list1[0])
	
	val map = hashMapOf("name" to "hj","sex" to "male")
   val map1 = map + ("name2" to "hah") //{"name":"hj","name2":"ha","sex":"male"}
   val map2 = map - "name"//{"sex":"male"}
   println(map2)
	```
	
1. ？运算符
	
	在java中，有时候为了避免出现空指针异常，我们通常需要这样的技巧：
	
	```
	if(rs!=null){
      rs.next()
      ...
	}
	```
	
   Kotlin可以使用?操作符达到同样的目的：
   
   ```
   rs?.next()
	```
	
1. Elvis操作符

	三目运算符通常以这种形式出现：
	
	```
	String displayName = name != null ? name : "Unknown";
	```
	
	Kotlin中可以简化为：
	
	```
	val displayName = name ?: "Unknown";
	```
	
1. 可空/非可空引用

	Kotlin中区分一个引用可以容纳null和不能容纳null。默认的引用是不可空的。
	
	```
	var a = "abc"
	a = null // 编译错误	```
	```
	
	需要使用?使其变为可空引用。
	
	```
	var b : String ? = "abc"
	b = null
	```
	
	如此，后续如果你调用a的任何方法都可以，但是调用b的会有编译错误。会强制去检查b是否为空
	
	```
	val l = if (b != null) b.length else -1
	```
	
	也可以使用?做安全调用
	
	```
	b?.length()
	```
	
	b不为空才会执行后续的操作。配合let可以执行其他非自身的操作。
	
	```
	b?.let({
		print("a")
	})
	```
	

最后，可以从各种资料中总结出Kotlin有以下几个特点：

1. 无缝与Java互操作

	可以直接使用Java库能够大大降低切换语言的成本。因此，如果想选择Kotlin，可以先试着用Kotlin写单元测试，之后等熟悉了此语言，可以进行混写。最后则完全切换到Kotlin。
	
1. 天然对协程的支持

1. 能够编译成原生二进制程序

1. 天然支持函数式编程和面向对象编程

1. 主流三方框架的天然支持