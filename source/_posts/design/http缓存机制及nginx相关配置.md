---
title: http缓存机制及nginx相关配置
date: 2016-11-8 20:11:37
categories: 
- 程序设计
tags:
- http
- nginx
---

# http缓存机制及nginx相关配置

浏览器http缓存，既是网页静态资源服务性能优化的一把利器，也是无数web开发者在工作之初谈之色变的一大难题。
在开发过程中我们极力避免缓存，但在生产环境中，我们又在想尽办法利用缓存。
所以了解浏览器缓存的机制，是一个优秀开发者绕不开的重要基础知识。

各种缓存的命中与否，说到底不过是几个与之相关的http header数据的匹配与校验。如果了解了每个相关header的意义与关系，那么就能将缓存策略运用自如。

## 缓存分类

![success response](http://7tszky.com1.z0.glb.clouddn.com/FgtfwfAjs5EBAEIa9KFi25acNk7d)

在浏览器的缓存模型中，一次成功（能拿到想要的数据）的请求，会有以上三种情况。

1. 200 from cache
    
    直接从本地缓存（有的文章称之为强缓存）中获取响应，返回200状态，chrome网络面板中的size项显示`(from cache)`。
    最快速，最省流量，因为根本没有向服务器发送请求。
    
2. 304 Not Modified
    
    在本地缓存没有命中的情况下，请求头中发送一定的校验数据到服务端，校验成功后服务器返回304 not modified表示资源未被修改，浏览器从本地缓存中获取响应，这种缓存方式通常被称为协商缓存。
    快速，发送的数据很少，只返回一些基本的响应头信息，数据量很小，不发送实际响应体。
    
3. 200 OK
    
    以上两种缓存全都失败，服务器返回完整响应。
    没有用到缓存，相对最慢。
    
## 本地缓存

本地缓存的使用，相当于基于之前服务器的response header，浏览器认为当前的缓存可以直接使用，于是不再与服务器进行任何交互而直接使用缓存的过程。

与本地缓存命中相关的http header：

1. Pragma

    这个是http1.0时代的遗留产物，该字段被设置为no-cache时（实际上现有的RFC标准标明只有这个可选值），会告知浏览器禁用本地缓存，即每次都向服务器发送请求。

2. Expires
    
    http1.0时代用来启用本地缓存的字段，expires值对应一个形如`Thu, 31 Dec 2037 23:55:55 GMT`的格林威治时间，告诉浏览器缓存实现的时刻，如果还没到该时刻，标明缓存有效，无需发送请求。
    
    但是这个方式有个很明显的问题，就是浏览器与服务器的时间是不能保证一致的，如果时间差距较大，那么会影响缓存管理的结果。
    
3. Cache-Control
    
    http1.1针对Expires时间不一致的问题，采取了一个十分聪明的设定，运用Cache-Control来告知浏览器缓存过期的时间间隔而不是时刻，那么即使具体时间不一致，也不影响缓存的管理。
    
    Cache-Control允许的值如下：
    * no-store
        
        禁止浏览器缓存响应，通常一些非常隐私的数据会启用这个值
        
    * no-cache
        
        不允许直接使用本地缓存，必须先发起请求和服务器协商。
        
    * max-age=delta-seconds
        
        告知浏览器该响应本地缓存有效的最长期限，以秒为单位。
    
    * 其他可选值不常见，以后遇到再补充
    
这三个字段主要用来告知浏览器的本地缓存管理策略，优先级为Pragma > Cache-Control > Expires。

## 协商缓存

当浏览器没有命中本地缓存，如本地缓存过期或者响应中声明不允许直接使用本地缓存，那么浏览器肯定会发起请求。

在http缓存模型中，即使浏览器向服务器发起请求，服务器也不一定要返回整个资源的实体内容。而可以返回协商结果：“浏览器，我的资源没有修改过，你可以直接使用你的本地缓存”。

很显然，服务器要判断浏览器的缓存是否可用，那么必须浏览器告诉服务器一些自己缓存的信息，所以协商缓存相关的header字段，必然是成对出现的。

1. Last-Modified
    
    格式：`Last-Modified: Tue, 08 Nov 2016 01:50:36 GMT`
    
    告诉浏览器资源的最后修改时间，相当于对资源进行了版本管理，至于这个时间怎么生成的，那是服务器的事儿，不在这里讨论。
    
    得知资源的最后修改时间后，客户端会将这个信息提交到服务器做检查，如果服务器验证出最后修改时间是一致的，那么表示该资源没有修改过，可以返回304状态。
    
    浏览器请求头中标记最终修改时间的header字段：
    
    `If-Modified-Since: Thu, 31 Mar 2016 07:07:52 GMT`

2. ETag
    
    即使我没有讨论服务器怎么生成最终修改时间，也可以相见，这个模式会存在不准确的问题：如果资源明明没有改变，但是Last-Modified发生了变化，那么就会返回整个资源实体。
    针对这个问题，http1.1还推出了ETag字段，服务器会根据某种计算方式（常见的如md5）给出一个标识符，这个标识符其实标记的是资源的实际内容。
    
    格式：`ETag:"58212f6c-22f23"`
    
    检测过程与Last-Modified类似，浏览器请求头中标记ETag的字段：
    
    `If-None-Match:"58212f6c-22f23"`
    
## 浏览器行为对缓存的影响
    
注意观察浏览器行为的开发者很容易发现，输入url访问与f5刷新，各个资源的请求速度好像不太相同。

常见的浏览器会将访问行为分为3种：

* 地址栏输入URL或书签访问
    
    按照正常策略使用缓存
    
* F5刷新

    跳过强缓存，但是会使用协商缓存

* Ctrl+F5

    跳过强缓存与协商缓存，直接加载资源实体
    
具体的实现方式可以想见的是发送不同的请求头。

## 缓存策略的选择

对大多数站点来说，以下内容是非常适合缓存的：

* 普通不变的图像，如logo，图标等
* js、css静态文件
* 可下载的内容，媒体文件

这些文件很少改变，适合长时间强缓存。

以下内容是做缓存时需要注意的，建议主要使用协商缓存的：

* HTML文件
* 经常替换的图片
* 经常修改的js、css文件

其中，js、css文件可以通过md5修改文件名的方式改变url来失效缓存，
即在文件内容变化后将main.95d21235.css改为main.1bcbf5de.css，由于url变化，所以不存在缓存的问题。

以下内容从来都不应该使用缓存：

* 用户隐私等敏感数据
* 经常改变的api数据接口

其中，后台rest api数据接口的如果需要引入缓存策略，必须要进行比较谨慎的规划，
将频繁改变的接口与基本不变的接口区分，并且在应用服务器中实现Last-Modified/ETag的生成机制以保证缓存不会造成错误的结果。

从这里延伸出去的话，理想情况下，一切网络资源都应该尽可能选择不同策略的缓存，但考虑到开发的成本与难度，这在现实中很难发生，因此应该尝试设置一些明智的缓存策略（最常见的就是给大量的静态图片设置缓存），以在长期缓存和站点改变的需求间达到平衡。

## nginx配置缓存策略

* 强缓存相关配置
    
    * add_header指令
    
        ```
        Syntax: 	add_header name value [always];
        Default: 	—
        Context: 	http, server, location, if in location
        ```
        给状态码2,3开头的响应添加响应头，如Pragma/Expires/Cache-Control，可以继承。

    * expires指令
        
        ```
        Syntax: 	expires [modified] time;
                    expires epoch | max | off;
        Default: 	
        expires off;
        Context: 	http, server, location, if in location
        ```
        expires为负值时，表示Cache-Control: no-cache;
        当为正或者0时，就表示Cache-Control: max-age=指定的时间(秒);
        当为max时，会把Expires设置为 “Thu, 31 Dec 2037 23:55:55 GMT”， Cache-Control 设置到 10 年;
        
* 协商缓存相关配置
    
    * ETag
        
        ```
        Syntax:     etag on | off;
        Default:    etag on;
        
        Context:    http, server, location
        ```
    * Last-Modified
        
        add_header指令，默认开启


## 参考文献

* http://imweb.io/topic/5795dcb6fb312541492eda8c
* https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn
* http://web.jobbole.com/84888/
* http://nginx.org/en/docs/http/ngx_http_headers_module.html
* https://linux.cn/article-5456-1.html
    
    
    
    
    
    
