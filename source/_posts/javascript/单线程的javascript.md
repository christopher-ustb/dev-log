---
title: 单线程的javascript
date: 2016-11-30 14:51:49
categories: 
- Javascript
tags:
- Javascript
- 多线程
---

# 单线程的javascript

## 一个有趣的死循环bug

在略去这个bug所有的外围细节后，我试图还原一个最小化的场景：

现在需要使用js将一个图片列表依次动态添加到页面上，但是有一个限定条件，在前一张图片没有加载完成之前，后一张图片不得抢先加载，必须等待前一张load事件触发。

这有何难：

```javascript
// 引入jQuery
// imgs 是图片数组
for (var i = 0, len = imgs.length; i < len; i ++){
    var img = imgs[i];
    $(document.body).append(img);
    var imgLoadSuccess = false;
    img.on('load', function() {
      imgLoadSuccess = true;
      console.log('img ' + img.attr('src') + ' load success.ready to load the next one.');
    });
    while(!imgLoadSuccess) {
        // wait until im load success
    }
}
```
从一个java开发者看来，这里的流程是这样的：

![多线程的javascript流程图](http://oh4zi4x28.bkt.clouddn.com//images/github-io/lebooks/%E5%A4%9A%E7%BA%BF%E7%A8%8Bjavascript%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

然而结果是，浏览器的javascript进入了一个可耻的死循环，并且图片也没有添加成功，load事件的回调方法也从来没有执行过。

## javascript为什么是单线程的

很多javascript开发者在接触js的第一天就会看到一句话：JavaScript是单线程运行的。

结合javascript语言作为浏览器脚本语言的最初用途，其实这个设定还是比较容易被接受的：
作为浏览器脚本语言，javascript的主要用途，就是与用户互动以及操作DOM元素，很明显如果多线程操作dom元素，将会大大提高问题的复杂度。

所以javascript在设计之初就定下了**单线程**这个核心特征。

随着js应用开发经验的日渐丰富，甚至promise编程的熟悉，这句话令一个java开发者更加百思不得其解，这么多的事件回调、ajax异步请求、promise编程，
都是在做异步编程的事情，明明是在**并发**，为什么说javascript是单线程运行呢？

## 参考文献

* [JavaScript 运行机制详解：再谈Event Loop - 阮一峰网络日志](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
* [深入理解JavaScript定时机制|Sina App Engine Blog](http://blog.sae.sina.com.cn/archives/2394)
* [细说JavaScript单线程的一些事 — 好JSER](http://hao.jser.com/archive/9038/)
* [[译]解析JavaScript的事件循环机制](http://lujingbo.me/posts/the-javascript-event-loop-explained.html)