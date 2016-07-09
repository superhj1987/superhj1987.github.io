---
layout: post
title: "搭建自己的github"
date: 2015-11-13 10:27:40 +0800
comments: true
categories: git
---

说起github，大家应该都是非常熟悉的。正是github的兴起，带来了开源的一个高潮，也诞生了无数优秀的开源项目。最最著名的Linux也在github上有了自己的repository。当然，github的核心技术git也是李纳斯的代表作。

记得几年前由于项目的需要，曾尝试自己去搭建一套git服务给项目组使用，折腾了好久，才总算搭建了一个基础的系统, 刚刚能用，权限控制都没有(<http://srhang.iteye.com/blog/1339110>)。但最终因为git的上手门槛有点高，还是选择了svn。后来随着github的兴起，git才如火如荼地在国内火了起来。许多大的互联网公司，也都开始把项目由svn转到git。但如果仅仅是搭建一个git服务，那么github这种网站提供的可视化ui带来的便捷却也不复存在了。对于一些小的有钱的团队，使用github的收费私人repository倒也是一种解决办法。但是，对于大部分公司来说，还是不会把公司内部的代码放到这种公共服务上的。这种需求场景下，就诞生了很多github的克隆实现，以方便部署内网的github。

<!--more-->

说到github的克隆实现，最出名的莫过于[gitlab](https://gitlab.com)。这是一个ROR的实现，应该是目前市面上最成熟的一个github克隆项目，功能也是最丰富的。它在原来最基础的git库管理、权限管理、用户管理的基础上又加入了诸如code review、ci等极大方便开发者的功能。基本把github克隆了一遍，还加上了github没有的功能。功能强大倒是强大，但是gitlab本身的部署非常复杂，需要安装很多依赖包和依赖组件，整个过程非常痛苦。虽然现在提供了集成安装包，但对操作系统又有要求，比如要求CentOS 6以上，CentOs 5的用户就享受不了这个便利了。。。一想到这么多事情，对于我这种懒人来说，还是没有选择gitlab。

在前东家的时候，git项目是从gitlab迁移到了gitbucket（其中的原因当然不仅仅是因为gitlab部署的繁琐）。说起[gitbucket](https://gitbucket.github.io/gitbucket-news/)，在百度上也搜不出什么信息来。能搜到的估计也就github库的地址和其他一些简单介绍。这个项目是日本的同行使用scala开发的一个github克隆。所以，抛开编程方面，对于java系的程序员还是比较友好的，部署也只需要把war包往tomcat之类的容器里一扔就ok。这个对比gitlab那可是天壤之别。而谈到二次开发，scala语言的学习曲线还是非常陡的，所以对比起来，貌似gitlab的二次开发相比较起来还是容易一些的（gitlab的二次开发没参与过，这里只是猜测）。当然，gitbucket是一个相对年轻的项目，对比gitlab，功能还显得比较单薄，bug也不少。我自己在使用的过程中，fix过几个小bug并提交到了项目的主分支，但是还有不少bug被公司的同事吐槽中（实在没时间专门fix这个）。另外，不得不说的是，gitbucket的markdown解析引擎，作者不知道什么原因从之前的一个叫pegdown的替换成了自己实现的markedj，让我们公司的一堆md文档都显示的各种乱，实在无语。。。不得不去clone了markedj的代码，fix了其中的一些bug，部署在我们内网的maven库里。bug虽然挺多，但比起gitlab来，还是喜欢gitbucket部署升级的简单（去官网下个war包，扔到tomcat里，恭喜你，你就拥有了自己的github）。当然，也有纯粹个人对jvm系语言的偏爱的原因^_^。

其实，除了gitlab和gitbucket还有很多github的克隆实现。这篇文章列出了很多并做了简单介绍:
<http://www.oschina.net/news/50222/git-code-platforms>

其实，开发工具这种东西，选择最适合自己以及自己团队的才是上上之选。