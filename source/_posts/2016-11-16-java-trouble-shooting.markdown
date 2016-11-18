---
layout: post
title: "JDK自带工具之问题排查场景示例"
date: 2016-11-16 22:21:34 +0800
comments: true
categories: java
---

## 目录

* [引言](#引言)
* [问题排查场景](#问题排查场景)
   * [获取正在运行的JVM列表](#获取正在运行的JVM列表)
   * [Java堆的DUMP](#Java堆的DUMP)
   * [分析类柱状图](#分析类柱状图)
   * [线程Dump](#线程Dump)
   * [运行Java飞行记录器(Java Flight Recorder)](#运行Java飞行记录器(Java Flight Recorder))
   * [后记](#后记)

最近看到了大量关于java性能调优、故障排查的文章，自己也写了一篇[Java调优经验谈](http://www.rowkey.me/blog/2016/11/02/java-profile/)。接着此篇文章，其实一直打算写写一些常用调优工具以及它们的惯常用法的。后来在<http://java-performance.info>这个站点上看到了类似的一篇博文，自我感觉很有指导意义。于是决定翻译+重组织一下此篇文章：[Java server application troubleshooting using JDK tools](http://java-performance.info/java-server-application-troubleshooting-using-jdk-tools/)。

## <a name='引言'></a>引言

在Java世界中，我们的很多开发工作从编码、调试到调优都在使用GUI工具。我们经常尝试在本地构建一套和线上一样的环境从而使得问题能够重现，进而使用我们常用的工具来排查定位故障。但不幸的是，很多情况下是无法在本地重现线上问题的。例如，我们没有权限获取真实客户端提交到线上服务端的数据。

由此，很多时候都是需要远程来排查线上服务器上发生的问题。但是如果单单只有一个JRE的话，你也不可能有合适的办法来进行排查。你需要JDK或者第三方的工具。有时候使用JDK提供的工具就是最可取的方案，毕竟，在线上环境使用第三方工具有时候会牵扯到权限的问题。

一般情况下，在线上环境安装JDK发布版本可以让排查进行地更高效。建议安装使用最新的Java7/8 JDK或者构建与线上JRE匹配的一些工具(原文作者不建议安装jdk的发布版本，而是建议根据实际需求逐渐地安装其中的工具)。

<!--more-->

## <a name='问题排查场景'></a>问题排查场景

### <a name='获取正在运行的JVM列表'></a>获取正在运行的JVM列表

为了开始排查工作，我们首先需要获取正在运行的jvm进程列表，包括进程id、命令行参数等。有时候仅仅这一步就可以定位到问题，例如，同样的app实例被重复启动在并发做同样的事情(破坏输出文件、重新打开sockets或者其他愚蠢的事情)。

使用**jcmd**不加任何参数即可获取jvm进程列表

	25691 org.apache.catalina.startup.Bootstrap start
	20730 org.apache.catalina.startup.Bootstrap start
	26828 sun.tools.jcmd.JCmd
	3883 org.apache.catalina.startup.Bootstrap start
	
使用**jcmd <PID> help**能够获取某个jvm进程其他可用的诊断命令。例如：

	[root@test-172-16-0-34-ip ~]# jcmd 3883 help
	3883:
	The following commands are available:
	VM.commercial_features
	ManagementAgent.stop
	ManagementAgent.start_local
	ManagementAgent.start
	Thread.print
	GC.class_histogram
	GC.heap_dump
	GC.run_finalization
	GC.run
	VM.uptime
	VM.flags
	VM.system_properties
	VM.command_line
	VM.version
	help
	
输入**jcmd <PID> <COMMAND_NAME>**可以运行一个诊断命令或者获取到参数错误信息。

	[root@test-172-16-0-34-ip ~]# jcmd 3883 GC.heap_dump
	3883:
	java.lang.IllegalArgumentException: Missing argument for diagnostic command	
	
通过**jcmd <PID> help <COMMAND_NAME>**你能够获取此诊断命令更多的信息。如下是**GC.heap_dump**命令的help。

	[root@test-172-16-0-34-ip ~]# jcmd 3883 help GC.heap_dump
	3883:
	GC.heap_dump
	Generate a HPROF format dump of the Java heap.

	Impact: High: Depends on Java heap size and content. Request a full GC unless the '-all' option is specified.

	Syntax : GC.heap_dump [options] <filename>

	Arguments:
		filename :  Name of the dump file (STRING, no default value)

	Options: (options must be specified using the <key> or <key>=<value> syntax)
		-all : [optional] Dump all objects, including unreachable objects (BOOLEAN, false)	
		
### <a name='Java堆的DUMP'></a>Java堆的DUMP

jcmd提供了输出HPROF格式的堆dump接口。运行**jmcd <PID> GC.heap_dump <FILENAME>**即可。注意这里的FILENAME是相对于运行中的jvm目录来说的，因此避免找不到dump的文件，这里推荐使用绝对路径。此外，也建议使用.hprof作为输出文件的扩展名。

在堆dump完成之后，你可以复制此文件到本地用VisualVM或者用jmc的JOverflow插件打开，进而通过分析堆的状况定位内存问题。

需要注意的两点：

- 还有很多可以打开hprof文件进行分析的工具：NetBeans, Elipse的MAT，jhat等等。用你最熟悉的即可。
- 同样可以使用**jmap -dump:live,file=<FILE_NAME> <PID>**来产生堆dump文件，但是官方文档标注了此工具为unsupported的。虽然我们绝大多数人都会认为JDK中unsupported的特性会永远存在，但是事实并非这样：[JEP 240](http://openjdk.java.net/jeps/240), [JEP 241](http://openjdk.java.net/jeps/241)。

### <a name='分析类柱状图'></a>分析类柱状图

如果正在排查内存泄漏问题，你可能想要知道堆中某种类型的存活对象数目。例如，某一时刻某些类应该只有一个实例(单例模式)，但是此类的另外一个或者多个实例却已经到了老年代，但是事实上它们不应该能够被GC roots访问到。

运行以下命令可以打印出类柱状图(同时也打印出存活对象的数目)：

	jcmd <PID> GC.class_histogram
	jmap -histo:live <PID>
	
输出如下：

		num     #instances         #bytes  class name
	----------------------------------------------
	   1:         37083       48318152  [B
	   2:        235781       22496784  [C
	   3:        103958       16069448  <constMethodKlass>
	   4:        482361       15435552  java.util.HashMap$Entry
	   5:        103958       14152480  <methodKlass>
	   6:          9576       11192168  <constantPoolKlass>
	   7:        186264       10430784  com.mysql.jdbc.ConnectionPropertiesImpl$BooleanConnectionProperty
	   8:        274109        8771488  java.util.Hashtable$Entry
	   9:          9576        7210152  <instanceKlassKlass>
	  10:          7972        6404256  <constantPoolCacheKlass>
	  11:        229637        5511288  java.lang.String
	  12:         48471        5428752  java.net.SocksSocketImpl
	  13:         21599        3859672  [Ljava.util.HashMap$Entry;
	  
这里的以byte为单位的占用大小是浅尺寸(shallow size)，并没有包括子对象的大小。其实这个事实很容易由char[]和String的统计数据注意到：这俩的实例数目是差不多的，但是char[]的占用大小要大很多，就是因为String并未包含下面的char[]的大小。

有了类柱状图信息，你就可以grep/search类的名字从而获取存活实例的数目。如果你发现某些类的实例数量比期望要大很多，你就可以做heap dump，然后用任意的heap分析工具来分析问题。

### <a name='线程Dump'></a>线程Dump

很多时候，应用会呈现出“卡在那里”的情形。这里有很多种卡住的状况：死锁、cpu密集运算。为了定位到问题所在需要知道线程在做什么、持有了什么锁等等。

Java中有两种锁：基于sychronized和Object.wait/notifyAll方法的原始锁以及java5引入的java.util.concurrent锁。这俩种锁的不同之处主要在于前者是限制在进入synchronied部分的地方的栈帧(stack frame)中的，并且会一直在线程dump中存在。后者却并不限制在栈帧中，你可以在一个方法中进入锁，在另一方法中解锁。因此，thread dump有时候并没有包含这些信息。尽管如此，还是应该使用thread dump来查看线程信息排查问题。

这里有三种方法可以打印应用的thread dump。

	kill -3 <PID> #仅限Linux平台
	jstack <PID>
	jcmd <PID> Thread.print
	
### <a name='运行Java飞行记录器(Java Flight Recorder)'></a>运行Java飞行记录器(Java Flight Recorder)

上面讲到的工具都是作为快速的查看诊断工具的。如果要深入分析问题，可以选择使用内置的Java飞行记录器:[Java Mission Control](http://java-performance.info/oracle-java-mission-control-overview/)。

运行JFR需要三步：

1. 创建一个包含了你自己配置的JFR模板文件。运行**jmc**, 然后**Window->Flight Recording Template Manage**菜单。准备好档案后，就可以导出文件，并移动到要排查问题的环境中。

2. 由于JFR需要JDK的商业证书，这一步需要解锁jdk的商业特性。

		jcmd <PID> VM.unlock_commercial_features

3. 最后你就可以启动JFR。
	
		jcmd <PID> JFR.start name=test duration=60s settings=template.jfc filename=output.jfr

	上述命令立即启动JFR并开始使用**templayte.jfc**的配置收集60s的JVM信息，输出到**output.jfr**中。
	
一旦记录完成之后，就可以复制.jfr文件到你的工作环境使用jmc GUI来分析。它几乎包含了排查jvm问题需要的所有信息，包括堆dump时的异常信息。

### <a name='后记'></a>后记

本文基本上是对英文原文的翻译，主要描述了几个常见问题的排查场景。

不得不说的是，JDK自带的工具是非常强大的。用好了这些工具其实已经足以应付绝大多数的Java问题排查场景。