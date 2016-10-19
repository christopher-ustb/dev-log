---
title: 消除SPA井号路由前缀
date: 2016-08-22 19:59:24
categories: 
- Javascript
tags:
- Javascript
- angularjs
---

# 消除基于angularjs构建的单页面应用（SPA）路由上的#前缀

1. $locationProvider.html5mode(true)
    
    $locationProvider.html5mode([mode])
        
    mode: 
    
    type: Object
    
    properties:
        
    * enabled-{boolean}-(default: false)，如果为true，会在支持history.pushState的浏览器依靠pushState来改变url，在不支持pushState的浏览器会变回#前缀的路径
    * requireBase-{boolean}-(default: true)，如果html5mode被使用，此参数指定是否需要呈现一个base标签。如果enabled requireBase都设置为true，并且没有base标签，那么在注入$location时angular会抛出错误
    * rewriteLinks-{boolean}-(default: true)，是否重写html5mode下的相对路径
    
    type: boolean
    
    只设置enabled
    
2. `<head>`中增加`<base href="/">`指定base路径

3. 刷新页面时给出get请求错误

    > Cannot GET /detail?workid=4
    
    3.1. 出现原因
    
    当第一次访问页面/detail，就是刷新页面后，浏览器无从得知这是否是一个真的url还是html5的state。
    所以浏览器直接去服务器寻址，并返回了404的错误。
    
    然而，如果根页面（index.html）以及所以的javascript代码全部加载完成，此时尝试导航到/detail，angular就能在浏览器尝试寻址服务器之前迅速地导航到该state。
    
    详见[stackoverflow](http://stackoverflow.com/questions/16569841/reloading-the-page-gives-wrong-get-request-with-angularjs-html5-mode)
    
    3.2. 此处的关键，在于将服务器上未解决的请求转移到SPA的根路径
    
4. mod rewrite

    [mod_rewrite](http://httpd.apache.org/docs/current/mod/mod_rewrite.html)是Apache web server的一个默认模块，篡改浏览器提交的url请求，并传递内容给浏览器。
    
    这个过程完全发生在服务端，浏览器毫不知情。结果页面就好像是源于提交的url（实际上不是），就像给源url带了一个面具。
    
    然而，这个功能跟经典的重定向是不同的，重定向只是简单地指挥浏览器跳转到另一个不同的服务器地址。
    
    由于mod rewrite有效地伪装了网站内容是从哪里、如何服务的，所以能提高网站安全性。由于静态化了网站路径，也能够用做搜索引擎优化。

5. [grunt plugin: connect-modrewrite](https://github.com/tinganho/connect-modrewrite)

    参考http://stackoverflow.com/questions/24283653/angularjs-html5mode-using-grunt-connect-grunt-0-4-5
    
    connect-modrewrite表达式的详细规则见于[github](https://github.com/tinganho/connect-modrewrite)

    在Gruntfile.js中connect中增加一个middleware:
        
        ```
        livereload: {
            options: {
              open: true,
              middleware: function (connect) {
                return [
                    // mod rewrite all request to /index.html
                  modRewrite(['^[^\\.]*$ /index.html [L]']),
                  connect.static('.tmp'),
                  connect().use(
                    '/bower_components',
                    connect.static('./bower_components')
                  ),
                  connect.static(appConfig.app)
                ];
              }
            }
          }
        ```

6. nginx config
    
    ```
    server {
        server_name yoursite.com;
        root /usr/share/html;
        index index.html;
    
        location / {
        try_files $uri $uri/ /index.html;
        }
    }
    ```
