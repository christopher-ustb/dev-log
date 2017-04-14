---
title: 特殊字符u2028造成的javascript错误
date: 2017-4-12
categories: 
- Javascript
tags:
- Javascript
- 编码
---

# 特殊字符u2028造成的javascript错误

## 症状

运用selenium RemoteWebDriver驱动phantomjs执行javascript脚本时，phantomjs抛出异常：
```
[ERROR - 2017-04-12T03:36:45.349Z] Session [feec3570-1f2a-11e7-8c8f-2d5cfc43fe3a] - page.onError - msg: SyntaxError: Unexpected EOF

  :262 in error
[ERROR - 2017-04-12T03:36:45.350Z] Session [feec3570-1f2a-11e7-8c8f-2d5cfc43fe3a] - page.onError - stack:
  evaluateJavaScript (phantomjs://webpage.evaluate():14)
  evaluate (:387)
  _executeCommand (:/ghostdriver/request_handlers/session_request_handler.js:404)
  _handle (:/ghostdriver/request_handlers/session_request_handler.js:142)
  _reroute (:/ghostdriver/request_handlers/request_handler.js:61)
  _handle (:/ghostdriver/request_handlers/router_request_handler.js:78)

  :262 in error
```

## 背景知识学习

### RemoteWebDriver

RemoteWebDriver是一个远程控制浏览器的规范接口。它提供了跨平台-语言无关的远距离查看与控制浏览器状态与行为的协议。

它提供了一系列的接口，来发现与操作网页中的dom元素，以及控制user agent的行为。它的原始意图是允许网站作者能够写出以一个单独的进程控制的自动化测试，也可以用作运行在浏览器内部的脚本来控制其他独立的浏览器。

比如：

* POST /session 新建会话
* POST /session/{session id}/url 修改浏览器的地址并转向
* POST /session/{session id}/refresh 刷新
* POST /session/{session id}/elements 查找元素
* POST /session/{session id}/element/{element id}/click 点击元素
* POST /session/{session id}/execute/sync 同步执行脚本

可以看出，RemoteWebDriver试图模拟人类用户对浏览器的大部分操作，以实现自动化测试的目的。

## selenium-remote

在了解了RemoteWebDriver的基本原理之后，selenium-remote的功能就已经呼之欲出了：封装对RemoteWebDriver的访问过程，管理session，简化开发。

在debug了一遍其源代码之后，总结其主要功能有：
* 利用命令行启动phantomjs进程，如何调用`GET /status`接口查看其是否启动完成
* 管理session，对应的WebDriver对象
* 组织http request
* 利用Apache http client发送请求
* 解析http response，包括成功结果与错误信息

## 问题追踪

1. 在了解了selenium-remote的基本原理后，怀疑其组织http request或解析http response有问题，将运行过程中的http request数据完全复制出来，拿到另外的发包工具（Fiddler）里发送后，得到如下错误：
    
    ```
    {
      "errorMessage": "Command failed without producing the expected error report",
      "request": {
        "headers": {
          "Accept": "application/json, image/png",
          "Connection": "Keep-Alive",
          "Content-Length": "136661",
          "Content-Type": "application/json; charset=utf-8",
          "Host": "localhost:26246"
        },
        "httpVersion": "1.1",
        "method": "POST",
        "post": "{\"args\":[],\"script\":"实际的js语句",
        "url": "/execute",
        "urlParsed": {
          "anchor": "",
          "query": "",
          "file": "execute",
          "directory": "/",
          "path": "/execute",
          "relative": "/execute",
          "port": "",
          "host": "",
          "password": "",
          "user": "",
          "userInfo": "",
          "authority": "",
          "protocol": "",
          "source": "/execute",
          "queryKey": {},
          "chunks": [
            "execute"
          ]
        },
        "urlOriginal": "/session/010c8a30-1c06-11e7-bf38-73a6c09b43c0/execute"
      }
    }
    ```
    基本看不出错误的原因，但是可以排除是类库组织http request的问题。

2. 查看phantomjs日志
    
    ```
    [ERROR - 2017-04-12T03:36:45.349Z] Session [feec3570-1f2a-11e7-8c8f-2d5cfc43fe3a] - page.onError - msg: SyntaxError: Unexpected EOF
    
      :262 in error
    [ERROR - 2017-04-12T03:36:45.350Z] Session [feec3570-1f2a-11e7-8c8f-2d5cfc43fe3a] - page.onError - stack:
      evaluateJavaScript (phantomjs://webpage.evaluate():14)
      evaluate (:387)
      _executeCommand (:/ghostdriver/request_handlers/session_request_handler.js:404)
      _handle (:/ghostdriver/request_handlers/session_request_handler.js:142)
      _reroute (:/ghostdriver/request_handlers/request_handler.js:61)
      _handle (:/ghostdriver/request_handlers/router_request_handler.js:78)
    
      :262 in error
    ```
    这个错误也相当模糊，看上去是js语法出现了问题。

3. 将运行的js代码复制到chrome控制台执行，这时候才第一次仔细观察要执行的js语句。

    ```javascript
    window.articles = [{"id":26802,"title":"猫驯记_12_小白","content":"<div class=\"show-content\">\n          <blockquote><p>猫也是人类最好的朋友，但它们绝不会屈尊承认这件事。<br> \u2014\u2014Doug Larson，作家<\/p><\/blockquote>\n<p>很久以前的那只小白猫似乎没有这么活泼。不知道是星座、血型还是性别原因\u2026\u2026反正小白就是比较沉稳。对，他叫小白。<\/p>\n<p>那时我在一个旧小区里租了套旧房子住，房东是个酷爱安利产品、并且将推广安利视为人生目标的老太太。安利老太太本来打算把这套房子按照学校宿舍风格装修\u2014\u2014在所有空间里都摆上双层铁架床，而且尽量省钱\u2014\u2014不过可能是高估了租客对双层床的偏好，最终剩下大堆床架床板，都胡乱塞在厅里。<\/p>\n<p>那套房间在一楼，空间不大，阳光不多，水泥地面冰凉。厨房和洗手间几乎是原生态，木门上涂了层黄漆，早已片片斑驳。<\/p>\n<p>小白第一次看见这套房子，抬头看了看我，然后往我手里吐了个泡泡。那天天气很热，他中暑了。<\/p>\n<p>当时小白很小，才和我手掌差不多长。尾巴细细瘦瘦，脑袋大大的，总是睁一只眼闭一只眼。脑袋上一小撮黑毛滑稽地竖着，身上的白毛还不够长，能看见粉红色的皮肤。<\/p>\n<p>小白在我手里不住吐泡泡，口水把下巴洇湿了一片。我也没什么经验，只能捧着他走来走去通风，帮他擦嘴角的口水。又倒了点水端到他嘴边，直到他开始喝水，才放心了下来。<\/p>\n<p>我去跟附近工地的工人们要了些沙子，又去买了点肉。回到家，发现小白不见了。<\/p>\n<p>那套房子基本上一览无余。两个旧沙发、一张折叠桌，剩下就是几张双层床。窗户和门都关好了，小家伙能跑到哪里去呢？<\/p>\n<p>不在厨房，也不在洗手间。唯一的可能，就是在那堆床板床架里。<\/p>\n<p>\u201c喵？\u201d我叫。<\/p>\n<p>没有回应。<\/p>\n<p>那堆床板床架结构很复杂，充满巧夺天工的随意感，像是实体化的混沌。我不敢动任何一块，因为那东西倒下来的方向和位置完全无法预测。我只能向每个缝隙里张望，点亮手机屏幕来照明，一边喵喵叫个不停。<\/p>\n<p>该死，我甚至连个手电都没有。<\/p>\n<p>我在考虑要不要去买个手电的时候，听到一声细细的\u201c喵~~\u201d。声音弱得让我怀疑自己幻听。<\/p>\n<p>\u201c喵~~\u201d又一声，细细弱弱怯生生的。是真的。<\/p>\n<p>我终于看见小白，在靠墙的角落里，紧贴着墙。两只大眼睛望着我，小下巴微微发抖。<\/p>\n<p>可是我够不着他。<\/p>\n<p>我坐在地上，和小白隔着一大堆杂物沟通。<\/p>\n<p>\u201c过来吧，来喝水？\u201d<\/p>\n<p>\u201c过来，来吃肉？\u201d\u2028<\/p>\n<p>小白惊恐地望着我。<\/p>\n<p>\u201c乖，来，要上厕所吗？\u201d我尽可能把声调再调整柔和一点。<\/p>\n<p>小白犹犹豫豫走近了两步，站住了。我伸出手，他又退后两步。<\/p>\n<p>我们就这样僵持了一下午。直到第二天早上，小白还缩在杂物堆里，偶尔轻轻叫一声。<\/p>\n<p>我下班回来，才看见杂物堆里探出一颗小脑袋，圆溜溜两只眼睛盯着我。头顶上的一撮黑毛还竖着，身上的颜色像是用久了的白麻布。<\/p>\n<p>\u201c小白？\u201d我慢慢走近。<\/p>\n<p>\u201c喵~~\u201d他回应，退后了一步，站住了。<\/p>\n<p>我伸手把他抱起来，他没躲开。小肚子扁扁的，细细的小肋骨根根分明。<\/p>\n<p>\u201c我们是室友啦。\u201d我对他说，\u201c请多多关照啊。\u201d<\/p>\n<p>小白的肚子贴在我手心，轻轻叫了一声。<\/p>\n\n        <\/div>"}];window.pageSizeId = 1;
    ```
    敏锐的观察到，其中有些很奇怪的Unicode字符。我好好的发送http请求，为什么要把我的字符转化为Unicode形式呢，看来还是组织http request的代码出了问题。

4. 跟踪selenium-remote源码，发现其使用的json化工具org.json:json会将某些字符转化为Unicode
    
    ```java
    default:
        if (c < ' ' || (c >= '\u0080' && c < '\u00a0') ||
                       (c >= '\u2000' && c < '\u2100')) {
            t = "000" + Integer.toHexString(c);
            sb.append("\\u" + t.substring(t.length() - 4));
        } else {
            sb.append(c);
        }
    }
    ```

5. 尝试缩小js范围，将每个Unicode替换为其实际字符，最后发现/u2028不是正常字符，而是一个Line Separator。反复对比后发现，这个字符是造成phantomjs执行js报错的直接导火索。

6. 找出直接原因，u2028为行分隔符，而javascript中字符串表达式不允许换行，从而导致执行错误。

## 问题总结

这里javascript对行分隔符的解析，可以看出一个动态语言的尴尬之处：
不同于Java等静态语言，由于这个javascript表达式是运行时生成的，所以解释器很难判断出，这个行分隔符是源代码中作者在试图换行，还是字符串的内容包含了航分隔符。
显然，这里javascript解释器认为脚本中包含了一个包含真正断行，而不是行分隔符的字符串表达式，于是判定为语法错误。

而这个数据的来源，可能是Java语言，在处理这个字符串的过程中，就不存在这个问题，于是可以顺利得将数据传递到这里交由javascript执行。

那么问题来了，javascript中，到底要如何表示多行字符串呢？

可行的方法：
1. 手动\n
```javascript
var multiple_lines = "举头望明月\n低头思故乡";
```
2. 结尾反斜线
```javascript
var multiple_lines = "举头望明月\
                      低头思故乡";
```
3. 字符串join('\n')
```javascript
var multiple_lines = ["举头望明月", "低头思故乡"].join('\n');
```

## 修复问题

在数据源头将u2048字符删去

## Reference

* [WebDriver](https://w3c.github.io/webdriver/webdriver-spec.html)

* [特殊字符_u2028导致的Javascript脚本异常 - 良村 - 博客园](http://www.cnblogs.com/rrooyy/p/5349978.html)

* [javascript的几种使用多行字符串的方式](http://jser.me/2013/08/20/javascript%E7%9A%84%E5%87%A0%E7%A7%8D%E4%BD%BF%E7%94%A8%E5%A4%9A%E8%A1%8C%E5%AD%97%E7%AC%A6%E4%B8%B2%E7%9A%84%E6%96%B9%E5%BC%8F.html)