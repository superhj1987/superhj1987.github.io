---
layout: post
title: "ShellShock这点事"
date: 2014-09-29 21:49:46 +0800
comments: true
categories: security
---

## 前言
在微博上看到最近安全界爆出了一个危害比之前的“心脏流血”（Heartbleed Bug）还要大很多的Bash代码注入漏洞：CVE-2014-6271 “shellshock”漏洞，然后随之而来一系列相关漏洞。详情可以看这些链接：[CVE-2014-6271](http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-6271) 、[CVE-2014-7169](http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-7169)、[CVE-2014-7186](https://access.redhat.com/security/cve/CVE-2014-7186)、[CVE-2014-7187](https://access.redhat.com/security/cve/CVE-2014-7187)、[CVE-2014-6277](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-6277)。世界上Linux服务器的占有份额是很大的，而bash又是Linux不可或缺的一个部分。可想而知，这个漏洞的破坏力有多大。这个从名字上就可以看出来，ShellShock是医学上的一种严重的疾病，中文叫做“弹震症”，指的是受到爆炸冲击后导致浑身颤抖、思维混乱等症状。这个命名很形象地反映了问题的严重性。

<!--more-->

## 漏洞的原理是什么

参照shellshock官网[https://shellshocker.net/](https://shellshocker.net/)，测试本机是否受这个漏洞的影响，先要执行一段shell代码：

<pre>
env x='() { :;}; echo vulnerable' bash -c ""
</pre>

如果发现有以下输出说明你系统受到这个漏洞的影响。

<pre>
vulnerable
</pre>

为什么会这样呢？看一下代码，首先设置一个环境变量x，x指向的是一个函数，这个函数仅仅有一句:;的代码，就是返回true。后面跟着的echo vulnerable，按说是不应该为执行的。后面的bash -c ...，这里使用bash命令开启了子shell，子shell会在启动的时候继承父shell的环境变量，于是在继承x这个变量的时候，就把echo vulnerable这行执行了。结果就是打印出了vulnerable。

官网上提到如果这一步就发现自己收到了影响，那么先update bash吧。

在升级完bash后，并非就高枕无忧了，又有人发现了更NB的利用这个漏洞的办法。执行下面的shell代码：

<pre>
env X='() { (shellshocker.net)=>\' bash -c "echo date"; cat echo ; rm -f echo
</pre>

如果这行代码，打印出了日期（可能会伴有一些错误），那么说明你仍然没有逃脱这个漏洞的影响。

<pre>
bash: X: line 1: syntax error near unexpected token `='
bash: X: line 1: `'
bash: error importing function definition for `X'
2014年 9月29日 星期一 21时04分30秒 CST
</pre>

update bash之后，只是让子shell继承父shell的时候不去执行后面的语句。但是这个代码变态之处在于它故意使用（shellshocker.net）=让shell报错，后面的>\则留在了缓冲区中，子shell继承到了>\,然后执行echo date，此时相当于下面的代码：

<pre>
>\
echo date
</pre>

\是命令换行的，于是就相当于>echo date，>是重定向符号，最后其实等价于date  > echo，这样最终把命令给执行了。

此外，官网还列出了其他的exploit，都是利用了子进程对环境变量的继承：

1. Exploit 3 (???)

	Here is another variation of the exploit. Please leave a comment below if you know the CVE of this exploit.
	<pre>
env -i X=' () { }; echo hello' bash -c 'date'
</pre>

	If the above command outputs "hello", you are vulnerable.

2. Exploit 4 (CVE-2014-7186)
	
3. Exploit 5 (CVE-2014-7187)

## 怎样利用这个漏洞

看了上面说的bash漏洞，那我们怎样来利用呢？举一个典型的列子，现在有很多网站是使用的apache运行在Linux系统上的，也是以子进程的方式来运行web程序的，其中用户端传来的HTTP_USER_AGENT、HTTP_HEADER等是会传递到子进程中的，而且这些变量是用户端任意可以指定的，如果按照开始讲的那样传递一个x过来，但是并不仅仅是echo一个字符串，那危害。。。可想而知。如下面的一个http请求（如果不是仅仅ping一个ip地址）：

<pre>
http-user-agent = shellshock-scan () http-header = Cookie:() { :; }; ping -c 163.com
http-header = Host:() { :; }; ping -c 163.com
http-header = Referer:() { :; }; ping -c 163.com
</pre>

除此之外，现在已经有利用这个漏洞攻击DHCP客户端、VoIP设备、Git版本控制系统、qmail等的成功例子了，可以说有Linux的地方就有shellshock的“用武之地”，包括Mac os。这篇文章总结了现在发现的各种各样的攻击方式：[http://www.fireeye.com/blog/uncategorized/2014/09/shellshock-in-the-wild.html](http://www.fireeye.com/blog/uncategorized/2014/09/shellshock-in-the-wild.html)

看到这里，可能有人会说：”让你们天天说Linux有多安全”。其实Windows也逃不开这个漏洞的危害，很多windows系统里都有bash环境，即使没有bash环境，只要你的系统使用了含有bash的组件（如负载均衡、CDN）也难逃shellshock的魔掌。

## 总结

修复Shellshock漏洞就像打地鼠，堵了一头另一头又冒出，修复一部分，很快就有其他的攻击方式出现，层出不穷，问题的关键其实还是在于bash在设计的时候对于环境变量的依赖。只要存在对环境变量的导出，那么攻击者就可以使用各种方式诱骗bash视其为命令，进行执行。


### 参考文章

* http://www.oschina.net/news/55694/shellshock-flaw
* http://news.cnblogs.com/n/504675/
* http://weibo.com/p/1005051401527553/weibo
* https://shellshocker.net/
