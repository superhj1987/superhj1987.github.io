---
layout: post
title: "如何读取ClassPath下的资源"
date: 2017-03-16 22:21:34 +0800
comments: true
categories: java
---

最近在写一些公共组件时，碰到了需要读取classpath下文件的场景。也突然想起其实之前有很多场景牵扯到读取类加载路径下的文件内容、路径，包括非jar包和jar包中的。总结一下，以备后续使用。

由于获取类路径下的资源文件的都是基于URL的，因此这里需要先讲述一下URL的概念。URL(Uniform Resource Locator)即统一资源定位器，指向互联网资源的指针，是一种具体的资源。其一般的形式，如：

`scheme:[//host][:port]/[path][?query][#fragment]`

其中scheme包括：http、https、file、jar等。

这里需要区分URL和URI。URI, Uniform Resource Identifier，统一资源标识符，用来唯一的标识一个资源。其一般形式：

```
[scheme:][//authority][path][?query][#fragment]

```
其中，authority为[user-info@]host[:port]

可见，URL是一种具体的URI，只不过其scheme是非空的，它不仅仅标识一个资源，也能定位一个资源（即通过url能够访问到这个资源），因此其必须是绝对地址，即使是相对url，其本质也是相对于某绝对url来讲的，也是一个绝对地址。而URI可以是绝对的也可以是相对的，只要能够标识即可。

此外，URL和URI的不同之处还在于前者不提供对标准RFC2396规定的特殊字符的转义，因此需要调用者自己对URL各组成部分进行encode。而java.net.URI则会提供转义功能。这两者可以通过URI.toURL()和URL.toURI()来互相转换。

这里需要指出的是，如果是想直接读取类路径下的资源的内容，那么使用下面的方法是万能的。

`ClassLoader.getResourceAsStream(String classpathRecourcePath)`

需要注意的是有时候jar包中的类并非和你应用的类使用的是同一类加载器(写intellij插件的时候就会存在这种问题)，这时候需要选择对应的ClassLoader。

此外，有些场景是需要获取到类路径下的资源路径信息的，可以选用以下三种方法：

- A: `ClassLoader.getResource(String classpathFilePath)`

	> A方法的加载过程类似`双亲委派机制`，当父加载器无法获取到资源时，自己才去尝试获取。但需要注意的是以`/`开头的资源是在类加载器目录下的资源，并非指的是当前应用的类加载路径下的资源。如果资源是位于classpath下的不要以`/`开头。
	
- B: `Class<?>.getResource(String classpathFilePath)`

	> B方法最终还是对A方法的调用。不同的是，A方法会对传入的路径参数做处理，并且会尝试去获取类加载器。
	- 当路径信息不以`/`开头时，A方法获取的是相对于当前类的相对资源
	- 当路径信息以`/`开头时，则获取的是当前应用类加载路径下的资源。

- C: `Class<?>.getProtectionDomain().getCodeBase.getLocation()`
	
	> C方法获取的是此类所处于的保护域的路径信息，当位于jar包中时，返回的是jar包的路径信息，非jar包则返回的是应用的类加载路径的地址。此方法的一个常见使用场景就是使用嵌入式jetty或者tomcat时对于webappBase的设置。

还需要提到的一点是：当你想使用File类来处理scheme为file的资源时，可以使用URL的getFile方法获取其path和query信息(URL的getPath方法返回的仅仅包含path部分)。但如果你的资源是位于jar包中的，那么获取到的URL信息是以`jar:file`开头的，并不能用此方式处理。