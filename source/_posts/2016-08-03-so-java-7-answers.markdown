---
layout: post
title: "[译]StackOverflow: 你没见过的七个最好的Java答案"
date: 2016-08-03 20:30:34 +0800
comments: true
categories: java translate
---

原文：<https://dzone.com/articles/stackoverflow-7-of-the-best-java-answers-that-you>

StackOverflow(后边简称so)发展到目前，已经成为了全球开发者的金矿。它能够帮助我们找到在各个领域遇到的问题的最有用的解决方案，同时我们也会从中学习到很多新的东西。这篇文章是在我们审阅了so上最流行的Java问题以及答案后从中挑出来的。即使你是一个有丰富经验的开发者，也能从中学到不少东西。

<!--more-->

## 分支预测

SO上最多投票的一个Java问题是：[为什么处理一个排序数组要比非排序数组快的多](http://stackoverflow.com/questions/11227809/why-is-it-faster-to-process-a-sorted-array-than-an-unsorted-array)。为了回答这个问题，你需要使用分支预测(branch prediction)。分支预测是一种架构，旨在通过在真实的路径发生前猜测某一分支的下一步来提升处理过程。

分支在这里即一个if语句。这样的话，如果是一个排序数组，那么分支预测将会进行，否则不会进行。[Mysticial](http://stackoverflow.com/questions/11227809/why-is-it-faster-to-process-a-sorted-array-than-an-unsorted-array/11227902#11227902)(so上的一个回答者)试图使用铁路和火车来简单介绍这个概念。假设你在铁轨连接处要决定火车要走哪条路，你会选择左边还是右边？你可以拦住火车，然后问司机该往那里，但是这样会让整个过程变慢。因此你只能去猜正确的方向，那么如何去猜呢？最好的办法就是通过观察目前这个火车每次经过时的路线，推测出正确的方向。

这就是分支预测：识别模式并使用它。

不幸的是，这个问题的提问者是分支预测失败的受害者。因为他的分支没有任何可以识别出的模式，所以预测出的行为是随机的。

## Java中的安全

另一个流行的Java问题是：[为什么在Java中有关密码的地方更加喜欢使用char[]而不是String](http://stackoverflow.com/questions/8881291/why-is-char-preferred-over-string-for-passwords-in-java)？其实原始的问题更加具体一些，就是问的在Swing中，password控件有一个getPassword方法(返回char[]而不是getText()返回的String)。

其实这里不用惊讶-这是一个安全问题。String是不可变的，意味着一旦它被创建了，那么你就不可能去修改它。这也意味着在GC之前，你对这些数据不能做任何处理。因此，只要有人能够访问你的内存，那么String就有可能被他获取到。

这也就是为什么要使用char数组。你可以显示地清除数据或者覆盖它。这样密码这种敏感数据即使GC还没有进行也不会再在系统留下痕迹。

## 异常

即使很多开发者倾向于忽略对受检异常的处理，SO上仍然有很多关于异常的问题。其中一个最流行的问题是：什么是NullPointerException，我该怎么处理它？对此，我们并没有感到惊讶，因为这个问题也是[在生产环境的Java应用中排名第一的异常](http://blog.takipi.com/the-top-10-exceptions-types-in-production-java-applications-based-on-1b-events/)。

实际上，当NullPointerException(或者其他exception)在系统出现的时候，我们可以发出一个告警。因为这种异常一般情况下都是业务代码逻辑有问题造成(笔者注)。

## 为什么这段代码使用随机字符串打印出了"hello world"

问题链接：<http://stackoverflow.com/questions/15182496/why-does-this-code-using-random-strings-print-hello-world>

这个问题给出了下面的代码，并打印出了"hello world"。

	System.out.println(randomString(-229985452) + " " + randomString(-147909649));
	
	public static String randomString(int i){
	    Random ran = new Random(i);
	    StringBuilder sb = new StringBuilder();
	    while (true)
	    {
	        int k = ran.nextInt(27);
	        if (k == 0)
	            break;

	        sb.append((char)('`' + k));
	    }

	    return sb.toString();
	}
	
其实，选择一组随机的整数并不是随机的。给定一个seed参数(在这个例子中是-229985452和-147909649), 那么每次随机，同样的seed则会产生同样的输出。

Random(-229985452).nextInt(27)产生的前六个数字：8, 5, 12, 12, 15, 0

Random(-147909649).nextInt(27)产生的前六个数字：23, 15, 18, 12, 4, 0

这样，最终输出的就是"hello world"。

## 为什么两个时间戳相减(in 1927)得出一个奇怪的结果？

问题链接：<http://stackoverflow.com/questions/6841333/why-is-subtracting-these-two-times-in-1927-giving-a-strange-result>

	public static void main(String[] args) throws ParseException {
	    SimpleDateFormat sf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");  
	    String str3 = "1927-12-31 23:54:07";  
	    String str4 = "1927-12-31 23:54:08";  
	    Date sDt3 = sf.parse(str3);  
	    Date sDt4 = sf.parse(str4);  
	    long ld3 = sDt3.getTime() /1000;  
	    long ld4 = sDt4.getTime() /1000;
	    System.out.println(ld4-ld3);
	}

按说上面的代码最后的结果应该是1，但实际的输出却是353。其实，这是一个时区的问题。1927年12月31号24:00，上海时间往回调整了5分钟52秒，因此"1927-12-31 23:54:08"发生了两次，Java将后面一次实例化成了本地的这个时间。因此和前一秒的差距成了353。

我们需要指出，如果你试着来运行这段代码，结果并不一定是353。[Jon Skeet指出了这一点](http://stackoverflow.com/a/6841479/5982245)，在时区数据库项目2014版中，这个改变的时间点改到了1900-12-31，因此成了344秒的差距。

## 无法被捕获的ChuckNorrisException

问题链接：<http://stackoverflow.com/questions/13883166/uncatchable-chucknorrisexception>

这里有一个很明显的问题：如果有exception被抛出，但是没有任何办法去catch，那么应用会崩溃吗？或者如这个问题所问：是否可以写一段Java代码让一个假设的java.lang.ChuckNorrisException无法被捕获。

答案是可以，但是这里有一个"但是"。你可以编译一段代码抛出一个ChuckNorrisException，但是在Runtime时动态生成一个并不继承于Throwable接口的ChuckNorrisException类。当然，为了让这个过程可以进行，你需要关闭掉字节码验证。[jtahlborn](http://stackoverflow.com/a/13883510/5982245)给出了完整的解决办法。

## 哈希表

哈希表是另外一个在SO上流行的问题系列。许多用户都想要知道所有集合类之间的区别，什么时候该使用哪种集合。

迭代顺序是主要考虑的因素。使用HashMap则忽略了所有的顺序信息，也就是获取元素的顺序和你插入元素的顺序是没有任何关系的；使用TreeMap则会得到一个排序好的迭代集合；使用LinkedHashMap则是一个FIFO的顺序。

如果你还是对这些感到困惑，这里有一个相关说明的图表可以[参考](http://zeroturnaround.com/wp-content/uploads/2016/04/Java-Collections-cheat-sheet.png)(Rebel Labs制作)。

## 总结

对于Java，其实关键的不在于你懂多少，而是在于你可以一直学到更多的东西。StackOverflow不仅在code上的一些问题可以帮助我们，也有助于我们回过头来去深入地学习一些我们已经知道的知识。