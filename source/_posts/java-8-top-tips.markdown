---
layout: post
title: "Java8 Top Tips[翻译]"
date: 2016-08-03 22:30:34 +0800
comments: true
categories: java translate
---

原文：<https://dzone.com/articles/java-8-top-tips>

本文包含了对于Java8的一些最佳实践，包括Stream和Lambda表达式的一些基础。

笔者已经使用Java8工作许多年，包括新的应用开发以及迁移旧的应用，感觉是时候总结Java8中一些有用东西的最佳实践。笔者个人不太喜欢“最佳实践”这个词，因为字面上传达了一种“one size fit all”的概念，当然，编码肯定不是这样的而是不同的场景有不同的解决方案。但是笔者觉得在如何使用Java 8让自己的生活变得更加容易上还是有一些特殊的经验值得分享的。

<!--more-->

## Optional

Optional是一个评价过低的特性，它可以显著的降低代码抛出NullPointerException的可能。它在边界代码(你正在使用的API或者你发布的API)中特别有用。

但是对于它的不适当的使用和设计很容易使一个小的变动影响到很多的类，或者降低代码的可阅读性。这里有一些如何更加高效使用Optional的建议。

### Optional应该仅仅用在返回类型中

不要用在参数或者域中。[阅读这篇博文](http://blog.joda.org/2015/08/java-se-8-optional-pragmatic-approach.html)可以看到如何正确使用Optional进行编码。幸运的是，IntelliJ IDEA可以打开inspections去检查你是否遵循了这些推荐规范。

![OptionalParamWarning.png](/images/blog_images/OptionalParamWarning.png)

要尽早在Optional出现的地方对它进行处理。IntelliJ IDEA会阻止Optional出现在你代码的各个地方，所以记住一定要在Optional出现的地方就对他进行处理。

![OptionalUseImmediately.png](/images/blog_images/OptionalUseImmediately.png)

### 不能简单地调用get()方法

Optional是用来表示这个值是有可能为空的，让你做好应对的准备。因此，很重要的一点就是在使用这个值之前务必要检查其是否存在。简单地调用get方法而不是先调用isPresent可能会导致产生空指针异常。幸运的是，IntelliJ IDEA再一次提供了对此种方案的检查。

![OptionalGetWithoutIsPresent.png](/images/blog_images/OptionalGetWithoutIsPresent.png)

### 更加优雅的方案

如下代码，isPresent和get当然能够解决这个问题。

![OptionalSimple.png](/images/blog_images/OptionalSimple.png)

但是这里有更加优雅的方式，你可以使用orElse来设置一个默认值但值是null的时候。

![OptionalOrElse.png](/images/blog_images/OptionalOrElse.png)

或者你可以使用orElseGet来设置当值为null的时候去调用的方法。虽然看着和前面的方案没有什么大的不同。但是提供的方法应该仅仅在需要调用的时候才被调动，那么当这是个很耗资源的方法时，那么使用lambda会带来更好的性能提升。

![OptionalOrElseGet.png](/images/blog_images/OptionalOrElseGet.png)

## 使用Lambda表达式

Lambda表达式是Java8最主要的卖点。即使你现在用不到Java8，你也应该对它有了一些基本的了解。下面讲述了一种新的方式使用Java编程，虽然这并不是一个“最佳实践”，仅仅是一个使用的指导。

### 保持简短

函数式编程对于长的lambda表达式是欢迎的，但是对于仅仅使用Java开发很多年的人发现编写短的lambda表达式会更容易一些。你甚至会想把表达式缩减到一行，会很容易把长的表达式重构成一个方法。