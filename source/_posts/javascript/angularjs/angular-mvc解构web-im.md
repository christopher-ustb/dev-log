---
title: angular-mvc解构web-im
date: 2016-10-20 14:16:51
categories: 
- Javascript
tags:
- 面向对象
- angularjs
---

# angular-mvc解构web-im

## 问题背景

之前喵书项目需要一个网页版即时通讯工具，即Web-im，只需要发文字信息跟图片，基本没有产品原型设计。考虑到服务器的分发能力，以及开发成本问题，最终选择了接入第三方即时通讯云服务器的方式，将开发工作的重点放在了前端上。

平心而论，这个任务其实是并没有什么困难的，无非是组织一下web im SDK的使用，难点都由第三方帮我们搞定了，总结这么一个文章对老鸟来说意义也很小。

但在实际开发中，由于缺乏全局性的设计和组件式的划分，最终造成这个小小的功能代码十分糟糕，忍无可忍必须重构。

以下主要记录一下思考的纲要图，另一方面，也简单做了一个mvc分层思想的demo。

## view显示层directive抽象

```html
<chat-window window-status="max/min/normal">
    <chat-contacts-bar></chat-contacts-bar>
    <div class="chat-timeline-wrapper">
        <chat-timeline active="">
            <chat-msg>aloha</chat-msg>
            <chat-msg>bye[bye]</chat-msg>
        </chat-timeline>
        <chat-timeline></chat-timeline>
    </div>
    <chat-sender-wrapper>
        <chat-emoji-sender></chat-emoji-panel>
        <chat-image-sender></chat-image-sender>
        <textarea></textarea>
    </chat-sender-wrapper>
</chat-window>
```
directive:
* chat-window 
    
    聊天窗体外壳，包含了聊天窗体的标题栏、外框等部分。
    
    其scope维护了一个窗体状态（最大化、最小化、正常，如果需要拖拽聊天窗体则也在此保存left/top等位置）的参数
* chat-contacts-bar
    
    左侧的联系人列表，每个列表项目包含了联系人头像、昵称等内容显示。
    
    其scope维护了自身的展开/收起状态。
* chat-timeline
    
    与某一个联系人的聊天时间线，包含了多个具体的聊天信息。
    
    其scope维护了自身的显示/隐藏状态，这个状态也能够被外层作用域所改变，比如用户点击联系人头像，即切换了当前聊天对象。
* chat-msg

    单条聊天记录，包含了发起者的头像、昵称以及聊天内容。
    
    这个指令应该能够自解析其内容，如表情及图片。
* chat-sender-wrapper
    
    整个聊天输入组件的外包装，包含了发送按钮。
* chat-emoji-sender
    
    聊天中使用到的emoji表情选择组件，包含表情按钮以及表情列表面板。
    
    其scope维护了面板的展开/收起状态。
    
    另外，其中由字符向表情图片的映射字典（如{'[laugh]': 'images/faces/laugh.gif'}）应该依赖于外部常量模块，以保证与chat-msg解析表情数据的一致性。
* chat-image-sender
    
    发送图片的组件，如果比较简单可以直接用html原生的element
 
## model数据层结构设计

此处数据层的设计，必须保存了的与业务逻辑耦合的数据，只标记组件的展示参数（典型的如组件的显隐）的参数不会在此处涉及，但不排除很多时候，组件的显示参数，是其背后业务数据发生变化的一种投射。

先定义几个基本原型(Class)：

* Contact
    * username 发消息时唯一标识
    * nickname 显示用的昵称
    * avatar 头像
    * initFromAppServer proto:Function 从数据库获取头像昵称信息
* GenericMessage
    * from 谁发的
    * to 发给谁
    * timestamp 发送时间
    * type 消息类型
    * isFromMe(me) proto:Function 是否是我发的
* TextMessage extends GenericMessage
    * text 文本内容
* ImageMessage extends GenericMessage
    * url 图片地址

此处假定将聊天数据模型设计在一个对象中，其属性及原型主要有如下：
* connection 维护了实际的聊天连接
* me proto:Contact
* chatting proto:Contact 当前正在与之聊天的联系人
* timelines proto:Object[]
    * contact proto:Contact
    * msgs proto:GenericMessage[]
* inputtingMsg proto:String
* isChatWindowActive proto:Boolean
* hasUnreadMsg proto:Boolean

该对象自身的行为：

* init() 初始化
    * initMe() 初始化me
    * initConnection() 初始化WebSocket连接
* receiveMsg(msg) 接收到信息
    * timelines.pushIfNotExisted() 如果需要的话添加新的timeline
    * timelines.findByContact(msg.from).msgs.push(msg) 找到对应的timeline并将msg加入其中
    * setChatting(msg.from)
    * setHasUnreadMsg() 根据窗体的active情况设置hasUnread标识
* toggleChatting(contact)
* sendMsg(msg)

   
## controller控制层方法定义

* chat-window controller
    
    作为整个聊天组件的最高层外包装，负责了所有与外部组件的交互动作，主要包括
    
    * 初始化im连接
    * 用户想与特定的人聊天或者只是想打开聊天窗体
        
        此处运用了一个angular的解耦思想：当事件发布者与事件监听者的父子关系无法确定时，借助顶级作用域发送事件，并且处理者自行订阅该事件，实现基于事件机制的松耦合。
   
* chat-contacts-bar controller
    
    * 点击列表单项切换当前聊天对象
    * 关闭单个聊天对象
    
* chat-sender-wrapper controller
    
    * 点击发送按钮发消息
    
* chat-emoji-sender
    
    * 选中了某个表情，将数据传递给wrapper
    
## service服务层定义全局事件

在全局中设置了一些全局事件传播的传播api，目前设计了如下：

* chat 打开聊天窗体，与某人聊天
* remindUnreadMsg 提醒监听者，出现了未读消息，执行比如显示小红点等页面行为
    