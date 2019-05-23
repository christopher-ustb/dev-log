---
title: ApacheHttpClient连接池源码分析
date: 2019-5-23 11:14:00
categories:
- Java
tags:
- Java
- 读源码
---

# ApacheHttpClient连接池源码分析

## 问题背景

公司基于apache http client为核心，实现了一个类似于[Zuul](https://github.com/Netflix/zuul)/nginx的网关系统。
为了保护后端被转发的集群，必须具有限流特性，即并发量控制。
我们知道apache http client自带了最大连接数的参数设置，在此细致解读一下其表现以及源码级别的实现方式。

## 连接池微观解读

执行堆栈查看得出，核心的处理代码在：`org.apache.http.pool.AbstractConnPool`，
作为一个抽象同步阻塞连接池，实现了接口

```java
public interface ConnPool<T, E> {
    // 租借连接
    Future<E> lease(final T route, final Object state, final FutureCallback<E> callback);
    // 释放连接
    void release(E entry, boolean reusable);
}
```

先看一下，`org.apache.http.pool.AbstractConnPool`类有哪些主要的属性：

```java
// 不同线程获取连接之间的竞态条件
private final Condition condition;
// 连接工厂
private final ConnFactory<T, C> connFactory;
// 保存route到连接池的map，apache http client可以设置参数setMaxConnPerRoute，按照route管理连接
private final Map<T, RouteSpecificPool<T, C, E>> routeToPool;
// 已租借出的连接
private final Set<E> leased;
// 可用的连接
private final LinkedList<E> available;
// 等待中的连接future对象
private final LinkedList<Future<E>> pending;
// route到连接数的映射
private final Map<T, Integer> maxPerRoute;
```

再来看一下我们最关心的`lease`方法:

```java
@Override
public Future<E> lease(final T route, final Object state, final FutureCallback<E> callback) {
    ...

    return new Future<E>() {

        private final AtomicBoolean cancelled = new AtomicBoolean(false);
        private final AtomicBoolean done = new AtomicBoolean(false);
        private final AtomicReference<E> entryRef = new AtomicReference<E>(null);

        @Override
        public boolean cancel(final boolean mayInterruptIfRunning) {
            ...
        }

        ...

        @Override
        public E get() throws InterruptedException, ExecutionException {
            try {
                return get(0L, TimeUnit.MILLISECONDS);
            } catch (final TimeoutException ex) {
                throw new ExecutionException(ex);
            }
        }

        @Override
        public E get(final long timeout, final TimeUnit timeUnit) throws InterruptedException, ExecutionException, TimeoutException {
            final E entry = entryRef.get();
            if (entry != null) {
                return entry;
            }
            synchronized (this) {
                try {
                    for (;;) {
                        // 死循环一直借到连接为止，除非出现被打断的异常
                        final E leasedEntry = getPoolEntryBlocking(route, state, timeout, timeUnit, this);
                        ...
                        entryRef.set(leasedEntry);
                        done.set(true);
                        onLease(leasedEntry);
                        if (callback != null) {
                            callback.completed(leasedEntry);
                        }
                        return leasedEntry;
                    }
                } catch (final IOException ex) {
                    done.set(true);
                    if (callback != null) {
                        callback.failed(ex);
                    }
                    throw new ExecutionException(ex);
                }
            }
        }

    };
}
```

可以看到，这个方法直接返回了一个与连接池关联的Future对象，所有的秘密都在这个匿名类中。
着重看一下`Future.get`:
除了处理一些前后回调钩子（onRelease/callback）外，主要将功能委托给了getPoolEntryBlocking方法

```java
private E getPoolEntryBlocking(
        final T route, final Object state,
        final long timeout, final TimeUnit timeUnit,
        final Future<E> future) throws IOException, InterruptedException, TimeoutException {

    Date deadline = null;
    if (timeout > 0) {
        deadline = new Date (System.currentTimeMillis() + timeUnit.toMillis(timeout));
    }
    this.lock.lock();
    try {
        final RouteSpecificPool<T, C, E> pool = getPool(route);
        E entry;
        for (;;) {
            ...
        }
        throw new TimeoutException("Timeout waiting for connection");
    } finally {
        this.lock.unlock();
    }
}
```

对对象加锁之后，根据route来获取`RouteSpecificPool`对象，可以参考一下上文提到的属性 `routeToPool`。

在这个for循环中，主要分为以下几段来获取连接：

1. 尝试获取空闲的连接，如果获取成功直接return。顺便关闭、清除了过期的连接。

    ```java
    for (;;) {
        entry = pool.getFree(state);
        if (entry == null) {
            break;
        }
        if (entry.isExpired(System.currentTimeMillis())) {
            entry.close();
        }
        if (entry.isClosed()) {
            this.available.remove(entry);
            pool.free(entry, false);
        } else {
            break;
        }
    }
    if (entry != null) {
        this.available.remove(entry);
        this.leased.add(entry);
        onReuse(entry);
        return entry;
    }
    ```

    关于`pool.getFree(state)`中的state：
    这是`ConnPool`接口中`lease`方法定义了的入参，可以传入任意一个Object对象，含义是用来表示一种特殊的状态（通常是安全秘钥、token等），来定位同样的连接。如果不需要这个支持，可以传null。

2. 缩容超出maxPerRoute的连接池

    ```java
    // New connection is needed
    final int maxPerRoute = getMax(route);
    // Shrink the pool prior to allocating a new connection
    final int excess = Math.max(0, pool.getAllocatedCount() + 1 - maxPerRoute);
    if (excess > 0) {
        for (int i = 0; i < excess; i++) {
            final E lastUsed = pool.getLastUsed();
            if (lastUsed == null) {
                break;
            }
            lastUsed.close();
            this.available.remove(lastUsed);
            pool.remove(lastUsed);
        }
    }
    ```

3. 如果还有空余容量，从工厂对象`connFactory`生产新的连接加入池。

    ```java
    if (pool.getAllocatedCount() < maxPerRoute) {
        final int totalUsed = this.leased.size();
        final int freeCapacity = Math.max(this.maxTotal - totalUsed, 0);
        if (freeCapacity > 0) {
            final int totalAvailable = this.available.size();
            if (totalAvailable > freeCapacity - 1) {
                if (!this.available.isEmpty()) {
                    final E lastUsed = this.available.removeLast();
                    lastUsed.close();
                    final RouteSpecificPool<T, C, E> otherpool = getPool(lastUsed.getRoute());
                    otherpool.remove(lastUsed);
                }
            }
            final C conn = this.connFactory.create(route);
            entry = pool.add(conn);
            this.leased.add(entry);
            return entry;
        }
    }
    ```

4. 阻塞线程，等待空闲连接

    ```java
    boolean success = false;
    try {
        // 如果Future已经取消，那么直接跳出等待，结束外层死循环
        if (future.isCancelled()) {
            throw new InterruptedException("Operation interrupted");
        }
        pool.queue(future);
        this.pending.add(future);
        // 使用condition阻塞线程，等待condition.signalAll()
        if (deadline != null) {
            success = this.condition.awaitUntil(deadline);
        } else {
            this.condition.await();
            success = true;
        }
        if (future.isCancelled()) {
            throw new InterruptedException("Operation interrupted");
        }
    } finally {
        // In case of 'success', we were woken up by the
        // connection pool and should now have a connection
        // waiting for us, or else we're shutting down.
        // Just continue in the loop, both cases are checked.
        pool.unqueue(future);
        this.pending.remove(future);
    }
    ```

    被唤醒的时机：
    * 其他Future被cancel，不跟你抢了
    * 其他线程release连接，归还了，但这也不表示能获取到连接，需要在`queue`排队

5. 检验超时

    ```java
    // 如果未成功获取到对象，且等待超时，那么跳出循环，throw超时异常
    // check for spurious wakeup vs. timeout
        if (!success && (deadline != null && deadline.getTime() <= System.currentTimeMillis())) {
            break;
        }
    }
    throw new TimeoutException("Timeout waiting for connection");
    ```


看到这里，这个连接池的微观层面的代码实现已经基本剖析完毕，如果还有细节问题，可以再对照源码查看。

这里比较巧妙地组合了Future/Lock/Condition/Queue（这里并没有使用并发集合类如`LinkedBlockingQueue`，而用LinkedList实现，猜测可能是历史遗留原因），实现了一个清晰安全的连接池，并且留下了许多可扩展可定制的参数与回调，值得学习参考。

## 连接池宏观设计

上一部分从代码层面解读了连接池的微观实现，这里我们再宏观看一下，ApacheHttpClient如何使用这个连接池。

`org.apache.http.impl.conn.PoolingHttpClientConnectionManager`类维护了`org.apache.http.impl.conn.CPool`属性，该类是`AbstractConnPool`的实现类，基本功能都来自继承。

ApacheHttpClient使用了责任链模式，链条上的executor：

1. `RedirectExec`: 负责处理重定向
2. `RetryExec`： 负责决定在io错误时是不是重试
3. `ServiceUnavailableRetryExec`： 负责决定非2xx响应是否重试
4. `ProtocolExec`： 负责处理http参数，构建请求体
5. `org.apache.http.impl.execchain.MainClientExec`：最后一个executor，负责实际的请求、响应转换，就是他从`PoolingHttpClientConnectionManager`中获取连接对象

什么时候释放连接？

1. 流被关闭

```java
public boolean streamClosed(final InputStream wrapped) throws IOException {
    try {
        ...
    } finally {
        releaseManagedConnection();
    }
    return false;
}
```

2. 流检测读到eof

```java
public boolean eofDetected(final InputStream wrapped) throws IOException {
        try {
            ...
        } finally {
            releaseManagedConnection();
        }
        return false;
    }
```

## ‘池’

*池*是编程设计中非常常用的一种模式，能够高效地复用对象，网络连接这种初始化成本较高的对象的池化，是最典型的场景。
由于许多类库的支持，开发者可能很少需要去重复造轮子自己实现对象池，但深入理解池的实现，会让我们对一些常见的表现能够有更精确的把握，甚至针对一些定制化场景进行优化与修改，设计更强大更高级的池。

ps： 文中httpcore版本 4.4.6，版本间细节差别应该不影响对设计的理解。
