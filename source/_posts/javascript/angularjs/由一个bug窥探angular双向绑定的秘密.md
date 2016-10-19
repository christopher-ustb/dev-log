---
title: 由一个bug窥探angular双向绑定的秘密
date: 2016-10-09 22:16:24
categories: 
- Javascript
tags:
- Javascript
- angularjs
---

# 由一个bug窥探angular双向绑定的秘密

## bug背景简述

我试图封装一个“一加一减的数字输入框”的指令，点击左边的减号可以将输入框里的数字减一，点击右边的加号可以加一，这样的控件在电商网站的订单页面选择商品数量时十分常见。

我的指令代码如下：
```html
<span>
  <a href="javascript: void(0);" ng-click="addValue(-1)">-</a>
  <input type="text" ng-model="value" ng-change="onValueChange()">
  <a href="javascript: void(0);" ng-click="addValue(1)">+</a>
</span>
```

```javascript
angular.module('app').directive('numberInput', function () {
  return {
    restrict: 'E',
    replace: true,
    scope: {
      value: '=',
      onValueChange: '&'
    },
    templateUrl: 'components/number-input/number-input.html',
    controller: function ($scope) {
      $scope.addValue = function (stepValue) {
        $scope.value = $scope.value + stepValue;
        $scope.onValueChange();
      };
    }
  };
});
```

scope隔离了两个对象：
* value
    
    '='双向绑定了内外层scope.value，使父级scope可以指定input的值，input被操作后的值也可以同步到父级scope。
* onValueChange

    '&'将外层函数传入scope，当value值发生变化时执行此函数
    
---    
我尝试使用这个指令：

```html
<number-input value="cart.quantity" on-value-change="updateQuantity(cart.quantity)"></number-input>
```

```javascript
$scope.updateQuantity = function(quantity) {
    console.log('quantity', quantity);
}
```

bug出现：

当input中是1，我点击加号，使cart.quantity加到2时，updateQuantity方法被执行，但是打印的结果是1不是2。

直观bug判断：

两个scope对象的隔离都是达到预期的，这个bug体现了value值的动态绑定晚于我们的预期，即在onValueChange执行之后才改变了父级value的值。

那么问题来了：

value的值到底是什么时候通过何种方式同步到父级scope中去的呢？

## angular如何实现双向数据绑定

`以下都是个人理解，源代码没撸通，有错误概不负责`

**如果类比java的23种设计模式，angular实现双向数据绑定的核心，是两个无处不在的“代理模式”和“观察者模式”**

### 由dom元素上的ng-model向scope中的属性绑定

angular在初始化dom树的时候，遍历所有带有ng-model属性的元素，并将其类似onChange的函数表达式取出来，、
用一个代理方法在执行了原有的逻辑之后，将ng-model的值赋予到scope[attributes[ng-model]]上去。

由于js中function也是对象，所以代码上不需要像java的代理模式那样手动或者使用动态字节码技术（如CGlib）生成一个子类来重写原有方法。

### 由scope中的属性值变化向dom树的元素内容绑定

angular在初始化dom树时，将带有ng-bind、ng-model、{{expr}}表达式的元素以及表达式内容，
保存在一个观察者序列中。如果仔细查看angular的scope对象，会发现其中__private__属性中有一个$$watcher数组，
这个就是观察者模式中的订阅者，随时准备被scope通知到表达式值发生变化。
至于变化后如何改变dom，那就是写普通的js操作dom，无须赘述。

那么，当某件事情发生时，反复遍历$$watcher中的表达式有没有变化，
由于这样的遍历属于脏检测，检查效率也常常被不使用双向绑定或者使用基于setter机制的绑定技术使用者所诟病。

而这个检测同步的过程，就是大名鼎鼎的**digest cycle**（消化循环？？）

有个问题困扰了我很久，那就是上文所说的“某件事情”到底是什么事情，angular如何得知scope中的属性发生了变化。
我甚至像某些误人子弟的博主一样，想到了是不是使用了定时任务轮询式检查。

然而angular的开发者是不可能这么僵化的，他们对网页中模型值变化的原因做了一个精彩绝伦的抽象与概括：

* dom事件，如点击、输入(ng-click)
* xhr网络响应($http)
* 浏览器Location变化($location)
* Timer事件($timeout)

于是angular只需要将这些事件用自己的模块封装起来，在事件发生时，触发digest cycle，那么$$watcher中的元素就会被改变，就实现了数据绑定。

这也解释了为什么在angular中使用很多jQuery插件时，经常出现双向绑定失败的情况，
就是因为这些插件以jQuery的方式，避开了上面列出的事件操作了dom树，根本就没有进入digest cycle，双向绑定不可能凭空发生。

当然了，angular还是会把钩子留给开发者，那就是$apply/$digest方法。

但是在finally会触发digest cycle的方法中手动调用$digest，就会抛出`digest cycle in progress`错误，因为不可能需要同时运行2个digest cycle来脏检测数据。

## 分析问题

于是，这个bug就很容易解释了：

当onValueChange执行时，digest cycle还没开始，父级scope的值还没有发生同步的变化，所以打1没打2。

## 解决问题

目前有2种思路：

* 将双向绑定的值改成一个对象类型，即共享同一个指针，实际的值是对象的属性，这样就不需要digest cycle，子scope中跟父scope是同一份数据。
* 将onValueChange的执行时机改成digest cycle之后，或者提前用$timeout(onValueChange)触发digest cycle
