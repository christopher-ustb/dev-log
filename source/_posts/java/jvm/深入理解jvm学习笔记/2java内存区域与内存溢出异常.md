---
title: java内存区域与内存溢出异常
date: 2016-10-02 9:16:24
categories: 
- Java
tags:
- java
- jvm
- 读书笔记
---

# java内存区域与内存溢出异常

### 运行时数据区域

* 程序计数器(Program Counter Register)

    字节码行号指示器，用来控制指令的前进：如分支、循环、跳转、异常处理、线程恢复等。
* java虚拟机栈(Java Virtual Machine Stacks)
    
    * 通常所说的栈内存。
    * 在方法执行时创建栈帧，保存了局部变量表、操作数栈、动态链接、方法出口等信息。
    * 其中局部变量表存放了编译期可知的各种基本数据类型（8大基本类型）、对象引用以及returnAddress类型
    * 局部变量表空间相对确定，可在编译期完成分配，运行期间不会改变局部变量表的大小。
    * 此区域主要会导致STackOverflow-栈深度超出jvm允许深度异常、OutOfMemoryError异常。
* 本地方法栈(Native Method Stack)
    
    主要供native方法使用
* java堆(Java Heap)
    
    * 存放对象实例
    * Xms与Xmx表示其最小-最大分配内存
   
* 方法区(Method Area)

    * 主要存储：
        * 类信息
        * 常量
        * 静态变量
        * 即时编译器编译后的代码
    * 常被称之为永久代，jvm规范未对这个区域的回收做强制规定
    
* 运行时常量池(Runtime Constants Pool)

    * 是方法区的一部分
    * 除了编译期的字面量与符号引用，运行期新的常量也能加入池中

* 直接内存(Direct Memory)

    * 不是jvm运行时数据区

* 线程
    * 线程共享：方法区、堆
    * 线程隔离：虚拟机栈、本地方法栈、程序计数器
    
### HotSpot虚拟机

1. 对象的创建
    
    1. 检查类是否已加载
    2. 为新生对象分配内存
        * 指针碰撞：内存是规整的，通过移动指针分界点来分配内存
        * 空闲列表：内存不是规整的
        * 采用哪种分配方式最终是根据垃圾收集器是否带有压缩整理功能导致java堆是否规整决定的
    3. 线程安全
        * 线程同步处理
        * 本地线程分配缓冲(Thread Local Allocation Buffer, TLAB)
    4. 初始化零值
    5. 设置对象头 
    6. 执行构造器方法<init>
2. 对象的内存布局
    
    * 对象头(Header)
        * 对象是哪个类的实例
        * 类的元数据信息
        * 哈希码
        * gc分代
        * 锁状态标志
        * 线程持有的锁
        * 偏向线程ID
        * 偏向时间戳
    * 实例数据(Instance Data)
    * 对齐填充(Padding)
3. 对象的访问定位
    * 句柄访问
        
        句柄：可简单理解为智能指针，由操作系统所管理的引用标识，封装了操作范围内的服务。

        最大的优势在于，reference中存储的是稳定的句柄地址而不是指针地址，所以当对象的物理位置被移动时，reference数据无需任何修改，而在GC过程中，对象的移动十分普遍。
    * 直接指针访问

        优势：节省指针定位的开销，速度更快。
        HotSpot虚拟机是采用此访问定位方式。

### OutOfMemoryError异常

* java堆溢出

    java.lang.OutOfMemoryError: Java heap space

    演示范例： 向ArrayList死循环添加对象

* 栈溢出
    * 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常
    * 如果虚拟机在拓展栈时无法申请到足够的内存空间，将抛出OutOfMemoryError异常

    演示范例： 死递归

* 方法区和运行时常量池溢出
    
    java.lang.OutOfMemoryError: PermGen space

    演示范例： 
    * 使用String.intern()将字符串纳入运行时常量池，无限循环导致溢出
    * 使用CGLib字节码技术动态生成Class并装载至内存，最终造成方法区溢出

    方法区溢出其实十分常见：CGLib(常见的spring aop)、jsp应用、OSGi应用等。
