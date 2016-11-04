---
title: tomcat的默认连接器
date: 2016-11-4 13:40:16
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# tomcat的默认连接器

本章介绍的“默认连接器”是指tomcat的默认连接器，在新的tomcat中已经被弃用，被Coyote取代，但是其中很多优化的思想，仍然是不错的学习工具。

## HTTP1.1新特性

主要引入了connection: keep-alive来使用持久连接，用块编码来切断不同请求，服务器不再立即关闭连接，使用同一个连接来下载所有的资源，减少建立/关闭Http连接的系统开销。

## Connector接口

```java
public interface Connector {
    // ---- properties
    Container getContainer(); 绑定相关联的servlet容器
    void setContainer(Container container);
    // ---- method
    Request createRequest();
    Response createResponse();
}
```

在模型设计上，Connector与Processor是一对多关系。

## HttpConnector类

org.apache.catalina.connector.http.HttpConnector实现了LifeCycle接口，会基于事件地被调用initialize(),start()方法。

### 创建ServerSocket

通过一个ServerSocketFactory工厂模式来创建ServerSocket实例

### 维护HttpProcessor实例

HttpConnector内部维护了一个java.io.Stack栈结构的HttpProcessor对象池，避免每次重新创建对象。
而每个HttpProcessor运行在自己的线程中，这样同一个HttpConnector实例就可以处理多个HTTP请求了。

处理request请求时，会根据如下规则维护HttpProcessor对象池

```java
// Create the specified minimum number of processors
// 一直创建到最小对象池容量以上
while (curProcessors < minProcessors) { 
    // 一直创建到最大对象池容量为止
    if ((maxProcessors > 0) && (curProcessors >= maxProcessors))
        break;
    HttpProcessor processor = newProcessor();
    // 将HttpProcessor实例入栈
    recycle(processor);
}
```

### 提供HTTP请求服务

```java
// Hand this socket off to an appropriate processor
HttpProcessor processor = createProcessor();
if (processor == null) {
    try {
        log(sm.getString("httpConnector.noProcessor"));
        socket.close();
    } catch (IOException e) {
    }
    continue;
}
processor.assign(socket);
```

如果由于对象池满了拿不到processor，那么跳出循环不做处理，并且关闭socket连接。
否则，交由processor处理该请求。

## HttpProcessor类

此处着重介绍“连接器线程”和“处理器线程”的异步实现。

HttpProcessor的LifeCycle接口实现的start()方法

```java
/**
 * Start the background thread we will use for request processing.
 *
 * @exception LifecycleException if a fatal startup error occurs
 */
public void start() throws LifecycleException {
    if (started)
        throw new LifecycleException
            (sm.getString("httpProcessor.alreadyStarted"));
    lifecycle.fireLifecycleEvent(START_EVENT, null);
    started = true;

    threadStart();
}
/**
 * Start the background processing thread.
 */
private void threadStart() {
    log(sm.getString("httpProcessor.starting"));

    thread = new Thread(this, threadName);
    // 连接器线程不存在时，处理器线程就没有意义了，所以设置为守护线程
    thread.setDaemon(true); 
    thread.start();
}
```
于是HttpProcessor实例就运行在自己的线程中了，再看一下在运行什么：

```java
/**
 * The background thread that listens for incoming TCP/IP connections and
 * hands them off to an appropriate processor.
 */
public void run() {
    // Process requests until we receive a shutdown signal
    while (!stopped) {

        // Wait for the next socket to be assigned
        Socket socket = await();
        if (socket == null)
            continue;

        // Process the request from this socket
        try {
            process(socket);
        } catch (Throwable t) {
            log("process.invoke", t);
        }

        // Finish up this request
        connector.recycle(this);

    }

    // Tell threadStop() we have shut ourselves down successfully
    synchronized (threadSync) {
        threadSync.notifyAll();
    }
}

/**
 * Await a newly assigned Socket from our Connector, or <code>null</code>
 * if we are supposed to shut down.
 */
private synchronized Socket await() {

    // Wait for the Connector to provide a new Socket
    while (!available) {
        try {
            wait();
        } catch (InterruptedException e) {
        }
    }

    // Notify the Connector that we have received this Socket
    Socket socket = this.socket;
    available = false;
    notifyAll();

    if ((debug >= 1) && (socket != null))
        log("  The incoming request has been awaited");

    return (socket);

}
```

可以看出，在没有新的socket之前，这个run()方法其实是一直在阻塞在第一行代码的。

这时候，再看看HttpConnector中调用的assign()方法

```java
synchronized void assign(Socket socket) {
    // Wait for the Processor to get the previous Socket
    while (available) {
        try {
            wait();
        } catch (InterruptedException e) {
        }
    }

    // Store the newly available Socket and notify our thread
    this.socket = socket;
    available = true;
    notifyAll();

    if ((debug >= 1) && (socket != null))
        log(" An incoming request is being assigned");

}
```

由于这时候available是false，那个wait()此时是不执行的，于是在将socket的引用赋给属性变量后，notifyAll()唤醒了所有等待中的线程。
于是run()方法中的process()终于开始处理socket连接，进行很多比较耗时的解析处理操作。

那么assign()方法中的wait()是在什么时候阻塞自己呢，也就是available是true到底代表了什么状态？
很明显，assign()方法是有synchronized修饰符的，也就是说，available肯定不是用来锁住这个方法本身的。

将await()/assign()方法成对观察，可以发现，这个available标志在为true时，assign()方法中的wait()方法其实卡住了调用它的连接器线程，因为这个时候，socket还没有被正确地处理。
于是就可以解释，为什么这里真正处理的socket引用，是一个局部变量而不是成员变量，因为这样，可以在当前socket被处理完成之前，继续接收下一个socket对象。
成员变量的socket对象引用，其实只是一个接受后的暂存处，并不是真正的处理socket时的引用。

---

未解之谜：

既然是暂存处，为什么不设置成集合类型（如队列）呢？

---

所以，这个available，并不是在表达当前的processor是不是可用，而表示this.socket这个暂存处是不是有未处理的socket：
如果有，表示暂存处还有任务没处理，assign()方法必须等待；如果没有，表示没有新的任务，await()方法必须等待。

其实不难看出，这里tomcat的开发者利用了available这个flag位，以及wait()/notifyAll()这对睡眠/唤醒方法，实现了一个精彩的异步编程，使得一个连接器对应多个处理器，将监听socket与处理socket放在了不同的线程中。

## 处理请求

1. 解析连接

2. 解析请求

3. 解析请求头

    使用字符数组避免代价高昂的字符串操作

4. 让servlet容器处理请求
