---
layout: post
title: "多activity中退出整个程序"
date: 2011-05-30 21:39:34 +0800
comments: true
categories: android
---

## 问题
多activity中退出整个程序，例如从A->B->C->D，这时我需要从D直接退出程序。

## 网上资料
finish()和system(0)都只能退出单个activity。杀进程等的等方式都不行~~~

## 解决问题

我们知道Android的窗口类提供了历史栈，我们可以通过stack的原理来巧妙的实现，这里我们在D窗口打开A窗口时在Intent中直接加入标志Intent.FLAG_ACTIVITY_CLEAR_TOP，再次开启A时将会清除该进程空间的所有Activity。

在D中使用下面的代码:
<pre>
Intent intent = new Intent(); 
intent.setClass(D.this, A.class);
intent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);  //注意本行的FLAG设置
startActivity(intent);
finish();//关掉自己
</pre>

在A中加入代码：
<pre>
@Override
protected void onNewIntent(Intent intent) {
// TODO Auto-generated method stub
super.onNewIntent(intent);
//退出
        if ((Intent.FLAG_ACTIVITY_CLEAR_TOP & intent.getFlags()) != 0) {
finish();
}
}
</pre>
A的Manifest.xml配置成android:launchMode="singleTop"

## 原理总结
一般A是程序的入口点，从D起一个A的activity，加入标识Intent.FLAG_ACTIVITY_CLEAR_TOP这个过程中会把栈中B，C，都清理掉。因为A是android:launchMode="singleTop"
不会调用oncreate(),而是响应onNewIntent（）这时候判断Intent.FLAG_ACTIVITY_CLEAR_TOP，然后把A finish（）掉。
栈中A,B,C,D全部被清理。所以整个程序退出了。