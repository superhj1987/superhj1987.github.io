---
layout: post
title: "Java后端工程师技能树"
date: 2016-06-17 22:59:44 +0800
comments: true
categories: java
---

此技能树借鉴自<https://github.com/geekcompany/full-stack-tree>，基本涵盖了一个Java后端工程师应该具备的技能，如有遗漏或者错误，敬请指出。

**PS**: 有同学指出tomcat不是web server是servlet容器，这一点图中的确不够严谨，因此更新了此技能树。其实tomcat引入了apr之后也是能够作为web server的(其实目前绝大多servlet容器都同时提供了web服务器的功能，只是性能之类的不好而已)，当然把tomcat直接做为web server的应用还不够广泛。目前，nginx+tomcat还是普遍的七层负载均衡构建集群的方案。此外，这里还牵扯到了servlet容器和应用服务器的一个区别，为什么tomcat只是servlet容器而不叫JavaEE应用服务器呢？这一点大家可以通过此链接了解一下:<http://www.cnblogs.com/maydow/p/4821249.html>。当然，对于互联网领域目前的Java后端体系，weblogic、webspher这些应用服务器基本绝迹，因此并没有写在此技能树中。

<!--more-->

[![java-skill-tree](https://raw.githubusercontent.com/superhj1987/pragmatic-java-engineer/master/book/server-tech/media/java-skill-tree.png)](https://raw.githubusercontent.com/superhj1987/pragmatic-java-engineer/master/book/server-tech/media/java-skill-tree.png)