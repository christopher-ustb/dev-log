---
title: 一次看似诡异的maven依赖导致NoSuchFieldError异常
date: 2016-10-27 16:33:40
categories: 
- Java
tags:
- java
- maven
---

# 一次看似诡异的maven依赖导致NoSuchFieldError异常

## 问题背景

在使用maven多项目构建的web project打成war包，部署到tomcat后，发现有一个Enum class，其中的部分枚举属性一直不能访问（编译肯定能通过，否则无法打成WAR包），程序抛出java.lang.NoSuchFieldError异常。

## java.lang.NoSuchFieldError的出现情况

我猜测很多人对这个异常是很熟悉的，java.lang.NoSuchFieldError意味着试图访问一个类、接口、枚举不存在的属性。

那么问题来了，为什么这样的“编译错误”没有在编译期被检测出来，而在运行期被当做一个异常抛出呢。

我们试图来还原一下最微型的犯罪现场：

```java
public class A {
	public static int a = 1;
}
public class B {
	public static void main(String[] args) {
		System.out.println(A.a);
	}
}
```

当我们javac编译java运行之后，一切正常。

现在我们删除A中的static int a，然后重新编译A。

很明显，B是不可能知道A发生了变化的，于是执行B的时候，会抛出java.lang.NoSuchFieldError异常。

本质上，java.lang.NoSuchFieldError通常反映的是class文件版本的不匹配，B和一个不与自己版本匹配的A一起协作，所以发生了异常。

## 真实出现的场景

但真实开发场景下，我们有各种各样的构建工具（我这里的maven）和集成开发环境（比如我的intellij idea），一个project是肯定会被当做一个整体进行编译与打包的，实际上不可能出现上文所说的情况。

所以，通常情况下，这个异常主要是在我们使用第三方类库的时候出现，最典型的就是当我们不用maven管理的情况下，引入纷繁复杂的spring jar包时，这个鲜艳的红字会在控制台显示100次。很容易理解，这就是因为jar包作为已经编译完成的class文件的打包版本，是无法得知与之协作jar包的情况的，他们也不可能作为一个整体进行构建与打包。于是ClassLoader只能无奈地抛出这个异常。

类似的还有ClassNotFoundException, NoSuchMethodError等

## 整体重新编译也解决不了的情况

即使你将整个项目作为整体重新编译，也有一种情况是解决不了的：

那就是这个class被放在了extension libraries或bootstrap libraries。由于ClassLoader的双亲委托模型，jvm会优先加载启动类加载器和扩展类加载器对应目录的class，所以应用类加载器的class根本没有机会被加载，那么无论如何重编译都不可能解决问题。

## 我的问题的困境

根据上面的理论，我依次检查了我的tomcat webapp目录下的lib目录中的的jar包，我甚至将其解压、反编译以确定是否出现了编译上的问题。

接着我检查了我的启动类加载目录与扩展类加载器目录中的jar包，很明显，这种情况下，不可能作死把我的class或者jar包放在这里。

## stackoverflow网友的启发

stackoverflow.com上的网友提示，很有可能是在应用中有多份全限类名完全一致的class。

于是我在抛出异常的语句之前，加了一句log，打印这个class的getResource("")结果。结果出人意料，这个class的资源位置并不是我想象的位置。

还记得吗，我在开头就说明了，我是运用一个maven多项目构建的web project。

在我的开发过程中，我曾经创建过一个module，姑且称之为base-module-old，其中包含了这个报错的类。
我将其打包、安装、部署之后，我重构了这个模块，将其变成了base-module-new。
但是base-module-old由于名字与base-module-new不一致，所以其文件永远留在了我的本地maven repository中。
同时，我的众多子module中，有一个module的依赖没有删除掉base-module-old，虽然这个base-module-old实际没有任何用处，但是每次maven打包都会将他复制到web的lib目录中，于是就出现了多个版本的同名class。
同时，由于这两个class其实都指向了相同的java源文件，于是IDE也没有发现任何问题，点击跳转时，十分顺利地会跳转到正确的java源代码，看上去一切OK。

由于相当于代码依赖路径中有两条不同的路径去加载该class，所以这个问题也具有很强的隐蔽性。

## 总结

这里主要涉及到jvm classloader的相关知识，以及在一个classpath中出现了同样全限定名的class的经验。

另外，多熟悉java api以及一定的反射知识，方便在运行时检验class对象的实际情况。

## references

[Latte Blogger](http://craftingjava.blogspot.jp/2012/08/javalangnosuchfielderror_9430.html)
[stackoverflow上的提问](http://stackoverflow.com/questions/40275739/weird-java-lang-nosuchfielderror-in-webapp-of-tomcat-build-with-maven1)