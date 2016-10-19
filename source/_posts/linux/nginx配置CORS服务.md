---
title: nginx配置CORS服务
date: 2016-08-16 21:16:24
categories: 
- Linux
tags:
- nginx
---

# nginx配置CORS服务

## 同源策略

### 含义
同源策略(same origin policy)是浏览器安全的基石，同源是指三个相同

* 协议相同(http不能访问https)
* 域名相同(miaoshu.me不能访问www.miaoshu.me)
* 端口相同(localhost:80不能访问localhost:8080)

### 目的

保证用户信息的安全，防止恶意的网站窃取数据。

### 限制范围

* Cookie、LocalStorage 和 IndexDB 无法读取。
* DOM无法获得。(iframe, window.open)
* AJAX不能发送。

## CORS

相比如jsonp(网页动态添加script标签，向服务器请求json数据)的只能发送GET请求的限制，CORS(Cross-Origin Resource Sharing)是跨域ajax请求的根本解决方案。

CORS需要浏览器和服务器同时支持。

实现CORS通信的关键是服务器。只要服务器实现了CORS接口，就可以跨源通信。

### 两种请求

* simple request

	满足以下条件

	method: HEAD/GET/POST

	http header不超出以下name:

		* Accept
		* Accept-Language
		* Content-Language
		* Last-Event-ID
		* Content-Type: 只限于application/x-www-form-urlencoded、multipart/form-data、text/plain

* not-so-simple request

### 浏览器处理流程

* simple request

	直接发出CORS请求

* not-so-simple request

	预检请求(preflight)

## nginx配置

如果服务不是属于开放平台服务，则需要先过滤请求域名

并在response header中增加Access-Control-Allow-*系列参数

```
	set $cors '';
	if ($http_origin ~* 'https?://(localhost|www\.miaoshu\.me|miaoshu\.me|vstar\.lebooks\.me)') {
	        set $cors 'true';
	}

	if ($cors = 'true') {
	        add_header 'Access-Control-Allow-Origin' "$http_origin" always;
	        add_header 'Access-Control-Allow-Credentials' 'true' always;
	        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
	        add_header 'Access-Control-Allow-Headers' 'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With' always;
	}

```

## 无论response code是多少都允许跨域

add_header后增加参数always(此参数必须在1.7.5以上的nginx，否则服务无法启动)

## 允许跨域发送Cookie

angularjs中的$http服务可全局指定ajax请求的是否发送credentials(angular版本1.1.1以上)

```javascript
	angular.module('app').config(['$httpProvider', function($httpProvider) {
	  $httpProvider.defaults.withCredentials = true;
	}])
```
---

## reference
[浏览器同源政策及其规避方法-阮一峰网络日志](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)

[跨域资源共享 CORS 详解-阮一峰网络日志](http://www.ruanyifeng.com/blog/2016/04/cors.html)

[W3C Recomendation](https://www.w3.org/TR/cors/)

[nginx](http://nginx.org/en/docs/http/ngx_http_headers_module.html)