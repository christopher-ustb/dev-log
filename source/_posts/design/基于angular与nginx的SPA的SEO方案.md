---
title: 基于angular与nginx的SPA的SEO方案
date: 2016-11-24 10:38:24
categories: 
- 程序设计
tags:
- SPA
- SEO
- nginx
---

# 基于angular与nginx的SPA的SEO方案

## 问题背景

出于动静分离的考虑，喵书项目选择了SPA(Single Page Application)，并且也从中尝到了很大的甜头：
* 用户体验好，响应速度快
* 前后分离，各司其职，并行开发
* 减轻服务器压力，主要输出数据
* 同一套后台代码可用于Web与app

同时，对于一个以生产内容为主的SPA，就面临了一个十分严峻的问题：SEO(search engine optimization)。

这个问题的本质其实是由于搜索引擎的爬虫不会执行网页的javascript代码，所有依靠javascript获取的数据、改变的dom元素都无法被收录，与SPA这种形式并没有关系，只是对SPA来说尤为突出。

## Googlebot的协议

google应该早就为开发者考虑到这个问题，所以googlebot（google搜索引擎的爬虫程序）为SPA开发者建立了一个协议，专门来解决javascript动态渲染的网站的爬取问题。

googlebot会将`http://localhost:3000/#!/profiles/1234`改成`http://localhost:3000/?_escaped_fragment_=/profiles/1234`爬取。

基于这个协议，只需要为带有`_escaped_fragment_`参数的请求提供不依靠javascript动态渲染的静态页面，就可以让爬虫拿到开发者希望被收录的数据。

而phantomjs就是这个帮助开发者由动态渲染生成静态文件的，基于Webkit的服务端javascript API。在这里起到的作用，就是模拟用户的浏览器，来生成与用户所看到的页面类似的html页面。

## 各种爬虫的兼容性难题

如果只需要支持googlebot的爬取，那么基于上面的协议就已经能着手解决问题。

但是，由于我们生活在一个特殊的国度，这里的搜索引擎不是google，而是百度、360、搜狗、有道等，以他们各自为战互相抹黑的态势，根本没办法保证他们遵守google的协议。

所以必须采用各种搜索引擎都能兼容的方案，才能做出本土化的SEO方案。

## 用mod_rewrite自定协议

检查http请求的user-agent，应该是最快的识别出搜索引擎网络爬虫的方法。

再运用服务端的某种设置，就完全可以自己制定一个类似于Googlebot escaped fragment协议的协议（推荐直接相同）。

实现原理如下：

1. 服务端检测http-request.user-agent是否是爬虫，如果不是，则正常返回用户页面；
2. 如果是网络爬虫，那么将经过一定规则改变的静态页面作为响应返回给爬虫。

由于我们使用nginx提供服务，nginx配置范例：

```
location / {
    if ($http_user_agent ~* '(Googlebot|Mediapartners-Google|AdsBot-Google|bingbot|Baiduspider|yahooseeker)') {
        rewrite ^(.*)$ /seo-site/$1;
    }
    ...用户浏览器服务配置
}

location /seo-site/ {
    ...静态页服务配置
}
```

那么问题就只剩下如何搭建这个提供静态页的服务，以及如何制定一个合理的rewrite规则来同时兼容SPA带#号的地址与HTML5模式不带#的地址了。

## Prerender





## 参考文献

[google ajax crawling](https://developers.google.com/webmasters/ajax-crawling/docs/specification)

[nginx配置location总结及rewrite规则写法](http://seanlook.com/2015/05/17/nginx-location-rewrite/)

[nginx配置范例](https://gist.github.com/thoop/8165802)