---
layout: post
title: "async源码分析"
date: 2014-08-22 16:08:28 +0800
comments: true
categories: 
---
最近在使用到node js的async库的时候，对其waterfall的实现感觉很奇妙，于是看了一下源码：
<pre>
	async.waterfall = function (tasks, callback) {
        callback = callback || function () {};
        if (!_isArray(tasks)) {
          var err = new Error('First argument to waterfall must be an array of functions');
          return callback(err);
        }
        if (!tasks.length) {
            return callback();
        }
        var wrapIterator = function (iterator) {
            return function (err) {
                if (err) {
                    callback.apply(null, arguments);
                    callback = function () {};
                }
                else {
                    var args = Array.prototype.slice.call(arguments, 1);
                    var next = iterator.next();
                    if (next) {
                        args.push(wrapIterator(next));
                    }
                    else {
                        args.push(callback);
                    }
                    async.setImmediate(function () {
                        iterator.apply(null, args);
                    });
                }
            };
        };
        wrapIterator(async.iterator(tasks))();
    };
 </pre>  
 
<!-- more --> 
  
开始先对参数进行了检查，判断tasks是否是一个function数组。然后使用了一个内部函数wrapIterator封装了实现。wrapIterator的参数带出了async.iterator函数：
<pre> 
 	async.iterator = function (tasks) {
        var makeCallback = function (index) {
            var fn = function () {
                if (tasks.length) {
                    tasks[index].apply(null, arguments);
                }
                return fn.next(); //这个地方有必要么？？？
            };
            fn.next = function () {
                return (index < tasks.length - 1) ? makeCallback(index + 1): null;
            };
            return fn;
        };
        return makeCallback(0);
    };
</pre>   
这个函数，其主要实现是其内部函数makeCallback。其功能就是迭代tasks，封装其中的每一个function,让其执行后返回下一个function,以此实现迭代。

接下来，再回到wrapIterator，此function是对iterator的封装。执行后返回的是一个匿名function。其明确的参数只有一个err。当err不为空的时候，直接执行callback function。否则从index为1开始取出参数列表，并把iterator的下一个function包装之后push到args中（如果没有下一个function了则push回调函数）。接下来，则执行当前的iterator，执行的参数是下一个iterator function（作为这一步的回调函数）以及参数（如果当前的iterator被调用时传递了其他参数）。这样在当前iterator中回调下一个iterator，依次迭代执行，直至执行完所有function和callback。