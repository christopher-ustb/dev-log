---
title: 请停止误用http-patch
date: 2017-4-14
categories: 
- 程序设计
tags:
- http
- 翻译
---

# 请停止误用http-patch

  修改http资源不是什么新鲜的话题。绝大多数现有的http或REST api都提供了修改资源的方式。
他们通常通过对资源使用PUT动词来提供这样的特性，要求客户端发送整个资源实体被更新后的值。
但是那就需要先对这个资源发一个GET请求，并且需要一种可以保证在这次GET调用与PUT调用之间不丢失任何更新的方法。
（译者注：如果在GET调用之后，其他的客户端修改了这个资源，那么后面的这次PUT请求就会将这个修改覆盖掉。）
并且，有的时候我们必须要考虑到，发送一个完整的实体，需要使用更多的带宽。
很多时候你并不想更新所有的值，只是想更新资源的一两个属性值。
所以PUT动词很可能不是部分更新————我们用来描述这种情况的术语 的正确的解决方式。
  
  另外一种解决方式是暴露你想要编辑的属性名称，然后使用PUT动词发送更新后的值。
在下面的例子中，user `123` 的 `email`属性被暴露出来：

```
PUT /users/123/email

new.email@example.com
```

这种方式看上去是个很不错的方法，可以自行决定暴露什么不暴露什么。
但是这也为你的API引入了许多复杂性（controller中更多的actions，路径定义，接口文档等等）。
然而，REST是允许这样做的，这也是个不坏的方式。但是有一个更好的方案：**PATCH**。

PATCH是RFC5789中定义了的http动词。
最开始想法是为了提出一个修改已存在资源的新的方式。
这个动词最大的问题在于很多人误解了它的用途。
**不！**
**PATCH** **完全不是**像第一段里描述的那样，关于用发送更新后的部分数值而不是整个资源的问题。
请现在就**停止**那样使用！这是错误的：

```
PATCH /users/123

{ "email": "new.email@example.org" }
```

这也不对：

```
PATCH /users/123?email=new.email@example.org
```

PATCH动词在其请求体中发送了必须运用在URI指定的资源上的**一系列变化**。
这一系列变化，包含了描述目前在服务器上的一个原始资源，应该怎样去修改，来产生一个新版本资源的指令。
你可以把它看做一个diff。

```
PATCH /users/123

[description of changes]
```

通过PATCH动词，这个变化集合必须自动地完全被应用，API必须从不提供一部分成功的修改结果。
（译者注：原子性，参考数据库事务进行理解。）
值得一提的是，PATCH的请求体，可以是与被修改的资源**不同的content-type**。
你必须使用一种为PATCH语义定义的media type，否则你就失去了这个动词的优势，可以使用PUT或者POST了。
RFC里这样描述：

> The difference between the PUT and PATCH requests is reflected in the way the server processes the enclosed entity to modify the resource identified by the Request-URI. In a PUT request, the enclosed entity is considered to be a modified version of the resource stored on the origin server, and the client is requesting that the stored version be replaced. With PATCH, however, the enclosed entity contains a set of instructions describing how a resource currently residing on the origin server should be modified to produce a new version. The PATCH method affects the resource identified by the Request-URI, and it also MAY have side effects on other resources; i.e., new resources may be created, or existing ones modified, by the application of a PATCH.

对于这个改变的集合，你可以使用任意的格式，只要清晰定义了其语义。
这也就是为什么用PATCH仅仅来发送更新后的值是不合适的。

RFC6902定义了一种使用于PATCH方式，来表达应用在JSON形式资源上的一系列操作 的JSON文档结构。

```json
[
    { "op": "test", "path": "/a/b/c", "value": "foo" },
    { "op": "remove", "path": "/a/b/c" },
    { "op": "add", "path": "/a/b/c", "value": [ "foo", "bar" ] },
    { "op": "replace", "path": "/a/b/c", "value": 42 },
    { "op": "move", "from": "/a/b/c", "path": "/a/b/d" },
    { "op": "copy", "from": "/a/b/d", "path": "/a/b/e" }
]
```

这里依赖与JSON指针（详见RFC 6901）来定位JSON文档，也就是HTTP资源中的具体值。

通过PATCH方式修改user123的email应该像这样：

```
PATCH /users/123

[
    { "op": "replace", "path": "/email", "value": "new.email@example.org" }
]
```

So readable, and expressive! Wonderful ♥
这才是PATCH的正确使用方法。如果成功了，状态码返回200。

RFC5261为XML爱好者定义了一个用XML PATH语言（XPath）选择器来更新XML资源的XML PATCH框架。

2014年末，RFC提倡了一种新的JSON Merge Patch格式来发送变化的集合。
这与只发送需要被修改的值的想法十分相似，但是由于`application/merge-patch+json` content-type使得它如此清晰明确。

总的来说，PATCH不是POST或者PUT方式的一个替代品。它是用来diff而不是替代整个资源。
PATCH请求体是与被修改的资源**不同的content-type**。它是描述应用在资源上的改变，而不是整个资源的表现。

从现在起，如果不能正确地使用PATCH，那么就不要使用。

值得一提的是，PATCH其实不是为真正的REST API设计的。
因为Fielding的论文没有定义任何一种**部分**修改资源的方式。
但是Roy Fielding自己说了，PATCH是他为原始的HTTP/1.1协议设计的，因为部分的PUT根本就不RESTful。


原文地址：http://williamdurand.fr/2014/02/14/please-do-not-patch-like-an-idiot/