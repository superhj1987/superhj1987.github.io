---
layout: post
title: "Kotlin语法简明指南"
date: 2018-12-08 19:29:34 +0800
comments: true
categories: java kotlin
---

Kotlin是Intellij IDEA的创造团队JetBrains发明的新一代JVM语言。虽然JVM上一次又一次出现新的语言叫嚣着取代Java，但时至今日，Java也开始吸纳其他语言的各种优势，其生命力依旧强盛，生态也越发强大。那么Kotlin的出现是又一次重蹈覆辙还是有其突破性的特性？

本文对其语法作了简要概括。

<!--more-->

**Kotlin版本：1.3.11**

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
	
	Kotlin中还能够直接通过表达式做为函数体来定义函数。
	
	```
	fun sum(a : Int, b : Int) = a + b
	```
	
	Kotlin中的函数和Java中的方法是一致的，但与Java不同的是，Kotlin中的函数可以属于任何类，文件当中直接定义则作为“包级函数”，和类的使用方式一致
		
1. 默认参数值

	函数的参数可以指定默认值。
	
	```
	fun getList(list: Array<String>, offset: Int = 0, size: Int = list.size) { …… }
	```

	不指定第2个参数调用方法时，offset参数取默认值0, size参数默认取第一个参数的size。
	
1. 可变参数

	函数的参数（通常是最后一个）可以用 vararg 修饰符标记：
	
	```
	fun printIntArray(vararg input: Int) {
        for (i in input) {
            println(i)
        }
    }
	```
	
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
	
1. if条件表达式

	Kotlin中支持if条件表达式。
	
	```
   val a = if(x > 0) 1 else 2
   fun maxOf(a: Int, b: Int) = if (a > b) a else b
	```

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
    
1. when

	 Kotlin中没有switch。提供when做分支条件选择。
	 
	 ```
	 when (x) {
        1 -> print("x == 1")
        2 -> print("x == 2")
        3, 4 -> print("x == 3 or x == 4")
        in 10..99999 -> print("x > 10")
        else -> { // 注意这个块
            print("x is neither 1 nor 2")
        }
    }
    
    when {
    	x.isOdd() -> print("x is odd")
    	x.isEven() -> print("x is even")
    	else -> print("x is funny")
	 }
	 ```
	 
	  when 既可以被当做表达式使用也可以被当做语句使用。如果它被当做表达式， 符合条件的分支的值就是整个表达式的值，如果当做语句使用， 则忽略个别分支的值。
 
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
	
	Map的遍历如下：
	
	```
	for ((k, v) in map) {
    	println("$k -> $v")
	}
	```
	
	Kotlin中的集合具有类似Java中的Stream的操作如filter、map、foreach等。
	
	```
	val positives = list.filter { x -> x > 0 }
	//val positives = list.filter { it > 0 }
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
	
1. 可空/非可空引用/函数返回值

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
	b?.let{
		print("a")
	)
	```
	
	同样的，对于函数参数以及返回值，默认也是非空的，只有加了?才允许传控制且要求做空值检测。
	
	```
	fun parseInt(str: String?): Int? {
    	// ……
    	if(str == null){
    		return null
    	}
    	
    	...
    	return ..
	}
	
	val r = parseInt(null)
	r?.let{
		print r
	}
	```
	
1. try with resources

	```
	val stream = Files.newInputStream(Paths.get("/some/file.txt"))
	stream.buffered().reader().use { reader ->
   		println(reader.readText())
	}
	```
	
1. 类

	- 无须public修饰符。文件名和类也没有任何关联。
	- 创建对象不需要使用new关键字

		```
		val test = Test()
		```
	- 对于类属性，默认会有get()和set()两个方法。直接访问属性或者给属性设置值都会调用这两个方法。

		```
		class Test {
	    	var counter = 0 // 注意：这个初始器直接为幕后字段赋值
	        get() {
	            println("getter")
	            return field
	        }
	        set(value) {
	            println("setter")
	            field = value
	        }
	
	
		}
		
		val test = Test()
		test.counter = 10
	   	println(test.counter)
		```
	- 主构造函数和次构造函数。Kotlin中一个类可以有一个主构造函数以及一个或多个次构造函数。主构造函数是类头的一部分：它跟在类名（与可选的类型参数）后。主构造函数里的参数如果用val或者var修饰则成为类的属性。如果类有一个主构造函数，每个次构造函数需要委托给主构造函数。主构造函数不能包含任何的代码。初始化的代码可以放到以 init 关键字作为前缀的初始化块（即时没有主构造函数，也会在次构造函数前执行）。

		```
		class Test(val counter: Int, val name: String = "test") {
		
			init{
				
			}

    		constructor(counter: Int, name: String, sex: String) : this(counter, name) {

    		}

		}
		
		val test = Test(10)
    	println(test.counter)
		```
	- Kotlin中引入了解构函数来对对象进行解构。

		```
		class Test(val counter: Int, val name: String = "test") {

    		operator fun component1() : Int{
        		return counter
    		}

    		operator fun component2() : String{
        		return name
    		}

		}
		
		val (counter,name) = Test(10)
		```	
		
		如此，也和map一样可以用在集合迭代中。
		
		```
		val testList = listOf(Test(1),Test(2))
    	for((k,v) in testList){
			...
    	}
		```
		
	- Kotlin中引入了数据类的概念。对于此种类，会默认根据主构造函数的属性生成equals()/hashCode()、toString()、componentN()、copy()这几个函数。

		```
		data class User(val name: String, val age: Int)
		```
		
	- Kotlin中提供了对象声明来实现单例模式。

		```
		object SingleInstance {
    		fun test(input: String) = println(input)
		}

		fun main(args: Array<String>) {
    		SingleInstance.test("hj")
		}
		```
	
	- Kotlin中提供了密封类来表示受限的类继承结构：当一个值为有限集中的类型、而不能有任何其他类型时。可以看做是枚举类的扩展。密封类需要在类名前面添加 sealed 修饰符。其所有子类都必须在与密封类自身相同的文件中声明。

		```
		sealed class DataType
		data class Card(val number: Double) :DataType()
		data class Timeline(val e1: DataType, val e2: DataType) : DataType()
		object Illegal : DataType()
		```
		
	- Kotlin的类中引入了伴生对象来声明静态方法、属性以及编译期常量（也可以在object中定义）。

		```
		class Test(val counter: Int, val name: String = "test") {

    		companion object {
        		const val TYPE = 1
        			val title = "haha"

        			fun testStatic(){
            		println("static method")
        		}
    	}
		
		```
		
	- 对一个对象调用多个方法。

		```
		class Test(val counter : Int){
		    fun test1(){

    		}

    		fun test2(){

    		}
		}

		
		val test = Test(1)
   		with(test){
      		test1()
        	test2()
    	}
		```
		
以上为Kotlin中的基本语法说明，其他诸如委托、lambda函数、协程、与Java互操作等可见<https://www.kotlincn.net/docs/reference/>。