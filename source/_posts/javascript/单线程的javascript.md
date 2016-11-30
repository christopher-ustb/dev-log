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

## 任务队列

javascript语言的单线程特性，就意味着所有的任务其需要排队。
那么各种各样的回调与异步编程是如何实现的呢？

稍有js异步编程经验的开发者其实都知道，通常需要使用异步编程的场景，都是与IO操作有关。
由于IO操作的速度与cpu运算有数量级上的差距，所以将这些IO操作返回结果之后的任务都设置为异步任务。

所以单线程js的异步运行机制如下：
1. 所有同步任务都在主线程上执行，形成了一个“执行栈”
2. 主线程之外，存在一个“任务队列”，只要异步任务返回了结果，就向“任务队列”中添加一个事件
3. 一旦“执行栈”中所有同步任务执行完毕，执行引擎就会去读取“任务队列”，将其队头的异步任务加入执行栈，开始执行
4. 主线程不停重复以上三步

![单线程js的异步运行机制](http://image.beekka.com/blog/2014/bg2014100801.jpg)

所以，上面那个死循环的代码其实是这样运行的：

![单线程的javascript流程图](http://oh4zi4x28.bkt.clouddn.com//images/github-io/lebooks/%E5%8D%95%E7%BA%BF%E7%A8%8Bjavascript%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## 利用事件机制实现异步编程

俗话说，在什么山上唱什么歌。
在单线程、使用任务队列的javascript语境下，就不应该再使用多线程同步的思维，类似于java中Object#await()/notifyAll()方法来锁住/唤醒线程，
而应该也运用事件机制，将某些需要唤醒的进程任务，用事件来触发。

为了不让数组中的图片在主线程中一次性全部添加进页面，将每个添加单张图片的行为封装成一个函数，并且利用图片加载事件来触发。
为了保存待添加图片的数组下标，必须将这个参数保存在函数之外的内存区域。

重构后的伪代码：
```javascript
var imgIndex = 0;
function appendImg() {
    if (imgIndex >= imgs.length) {
        return;
    }
    var img = imgs[imgIndex];
    $(document.body).append(img);
    img.on('load', function() {
        console.log('img ' + img.attr('src') + ' load success.ready to load the next one.');
        imgIndex ++;
        appendImg();
    });
    console.log('img ' + img.attr('src') + ' append to document. wait util loaded.');
}
appendImg();
```

## 与多线程实现方式的对比

将之前运用多线程思维的bug版代码与重构后的代码对比，可以看出其中关键性的区别，在于保存了图片数组的当前被处理的元素下标。

这个下标，其实相当于循环中的局部变量。
对应于多线程语言如Java在单CPU下的线程切换，其实这个下标就是方法执行的栈桢。
线程切换时，执行引擎必须将这个失去CPU的线程的栈桢保存起来，以便下次这个线程重新获得CPU时间的时候，能够把这个栈桢中的数据取出来，重新恢复上下文的执行环境。

而这里的javascript单线程实现方式，也可以类比为：
添加图片元素的线程appendImg()方法，通过结束代码的方式放弃CPU使用权，将栈桢中的数据，也就是数据下标保存到不被GC的window对象中。
下次appendImg()方法重新被任务队列唤醒，获取到CPU时，可以直接从window对象中获取上下文环境，继续执行添加任务。

某种程度上说，我这里是在用java的思想，实现本应由底层实现的线程调度。

对于我需要实现的排版算法，基本思路已有蓝图，对于习惯了多线程的开发者来说，利用单线程的事件机制来实现异步编程主要要处理好三件事情：
* 线程通过代码执行完毕来放弃CPU时间

    在js中，是绝不会出现阻塞主线程的方式的，因为根本没有主线程，js只有一个线程。
    如果希望代码停止执行，那么必须让同步任务执行完毕。
    
* 线程通过注册事件来给自己定义下次获取CPU时间的时机

    如果希望线程在某种情况下被唤醒，那么可以通过注册事件（定时器本质上也是一种事件：时间到了），将自己作为回调函数保存起来。
    
* 栈桢数据的保存与恢复

    为了下次线程重新获取CPU时能够正常执行，必须将线程内部的局部变量情况--栈桢自行保存，下次再进行恢复。
    在本文的最小化demo中，只有一个循环的逻辑，所以栈桢结构非常简单，只是一个简单的数组下标。
    但是在真实的方法层层调用、各种逻辑分支的控制情况下，其实上下文执行环境的数据必然是一个先进后出的栈的结构，如何妥善地保存与恢复这些数据，可以参考jvm底层原理。
    
## 致歉

本文完全从一个习惯了多线程运行的java开发者的视角去思考javascript异步机制，js原住民请忽略。

## 参考文献

* [JavaScript 运行机制详解：再谈Event Loop - 阮一峰网络日志](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
* [深入理解JavaScript定时机制|Sina App Engine Blog](http://blog.sae.sina.com.cn/archives/2394)
* [细说JavaScript单线程的一些事 — 好JSER](http://hao.jser.com/archive/9038/)
* [[译]解析JavaScript的事件循环机制](http://lujingbo.me/posts/the-javascript-event-loop-explained.html)