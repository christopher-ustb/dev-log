---
title: 一个简单的Web服务器
date: 2016-11-3 10:57:15
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# 一个简单的Web服务器

## 概述

相信了解一定socket知识的开发者都应该可以想象得到，即使tomcat的功能再怎么丰富多彩绚烂多姿，其最最基础的第一行代码仍然不过是普普通通的socket编程。
至于什么“容器”、“连接器”不过发展之后的模块化划分。

## HTTP

http协议是建立在在可靠tcp连接之上的一个基于请求-响应的网络协议。

其基础的约定，就是用一些回车换行、空格之类的特殊字符来将网络传输的字符串进行功能的划分，于是就有了所谓的请求头-URI-协议版本号、请求头、请求实体，协议-状态码-描述、响应头、响应体，其本质上不过是些按照一定标准格式书写的字符。

但话说回来，人类社会的共同协作，不正是基于这些想象中的共同标准吗。

## Socket类

这个鬼名词被翻译成了“套接字”，如果评选计算机专业十大糟糕翻译，它肯定是第一名的强力候选。

Socket是数据传输的端点，使得计算机之间可以通过网络读写数据。

java.net.Socket这个类类似于客户端，主要用来请求数据。

## ServerSocket类

java.net.ServerSocket相当于服务端，主要用来等待、处理客户端的通信请求。

## 应用程序

`为了节省篇幅文章中全部都是伪代码`

HttpServer
```java
public class HttpServer {
    public static void main(String[] args){
        HttpServer server = new HttpServer();
        server.await();
    }
    
    public void await() {
        ServerSocket serverSocket = new ServerSocket(...parameters);
        while(true){
            Socket socket = serverSocket.accept();
            InputStream input = socket.getInputStream();
            OutputStream output = socket.getOutputStream();
            // 将input/output即对应了request response
            ...
        }
    }
}
```

Request/Response

根据http协议解析输入/输出流中的字符串


