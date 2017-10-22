---
title: Android优化--线程池
date: 2017-10-21 23:20:21
tags: Android
categories: Android
---

## 为什么使用线程池

在Java中，执行异步大多使用两种操作，一是继承Thread类，而是实现Runnable接口。其中使用Runnable方法可以处理同一资源。而Thread类创建的线程则是各自处理。一般情况下，我们是这么实现的:

```
new Thread(new Runnable() {
            @Override
            public void run() {
                //do sth .
            }
        }).start();
```

这样实现是没什么问题，任务结束后会GC。不过如果一个程序中很多地方需要大量的线程处理任务，那么这样处理会让系统性能降低。对于Android来说，这样做有如下两个弊端

* 线程频繁的创建和销毁需要时间，导致了性能缺失。频繁的GC会让内存发生变动，这样会使手机变卡
* 大量的线程执行急剧占用内存和CPU

因此，我们需要一个合理的机制去管理这些线程，优化性能。所以提出了个线程池的这个概念。
<!--more-->
## 线程池

Java里面线程池的顶级接口是 Executor，不过真正的线程池接口是 ExecutorService， ExecutorService 的默认的实现是 ThreadPoolExecutor；如果我们直接用ThreadPoolExecutor来创建线程池的话，配置起来那是相当麻烦，看看这个构造：

```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {...}
```

在此先介绍下这么几个参数

1. corePoolSize：线程池的核心线程数，一般情况下不管有没有任务都会一直在线程池中一直存活，只有在 ThreadPoolExecutor 中的方法 allowCoreThreadTimeOut(boolean value) 设置为 true 时，闲置的核心线程会存在超时机制，如果在指定时间没有新任务来时，核心线程也会被终止，而这个时间间隔由第3个属性 keepAliveTime 指定。

2. maximumPoolSize：线程池所能容纳的最大线程数，当活动的线程数达到这个值后，后续的新任务将会被阻塞。

3. keepAliveTime：控制线程闲置时的超时时长，超过则终止该线程。一般情况下用于非核心线程，只有在 ThreadPoolExecutor 中的方法 allowCoreThreadTimeOut(boolean value) 设置为 true时，也作用于核心线程。

4. unit：用于指定 keepAliveTime 参数的时间单位，TimeUnit 是个 enum 枚举类型，常用的有：TimeUnit.HOURS(小时)、TimeUnit.MINUTES(分钟)、TimeUnit.SECONDS(秒) 和 TimeUnit.MILLISECONDS(毫秒)等。

5. workQueue：线程池的任务队列，通过线程池的 execute(Runnable command) 方法会将任务 Runnable 存储在队列中。

6. threadFactory：线程工厂，它是一个接口，用来为线程池创建新线程的。

7. handler：通常叫做拒绝策略，1、在线程池已经关闭的情况下 2、任务太多导致最大线程数和任务队列已经饱和，无法再接收新的任务 。在上面两种情况下，只要满足其中一种时，在使用execute()来提交新的任务时将会拒绝，而默认的拒绝策略是抛一个RejectedExecutionException异常。

参考链接：http://www.jianshu.com/p/b8197dd2934c

还好，Java官方提供了一个Executor的框架，其核心是Executors类，这个Executors里面调用的就是 ThreadPoolExecutor。摘取这个类的某一方法:

```
 public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
...
```
在源码中我们可以看到，Executors直接帮我们构造了线程池。Executors这个类是用工厂模式设计的，它里面封装好了众多功能不一样的线程池，从而使得我们创建线程池非常的简便。我们主要使用的是这么几个池：

1. newFixedThreadPool() ： 
该方法返回一个固定线程数量的线程池，该线程池中的线程数量始终不变，即不会再创建新的线程，也不会销毁已经创建好的线程，自始自终都是那几个固定的线程在工作，所以该线程池可以控制线程的最大并发数。 
栗子：假如有一个新任务提交时，线程池中如果有空闲的线程则立即使用空闲线程来处理任务，如果没有，则会把这个新任务存在一个任务队列中，一旦有线程空闲了，则按FIFO方式处理任务队列中的任务。

2. newCachedThreadPool() ： 
该方法返回一个可以根据实际情况调整线程池中线程的数量的线程池。即该线程池中的线程数量不确定，是根据实际情况动态调整的。 
栗子：假如该线程池中的所有线程都正在工作，而此时有新任务提交，那么将会创建新的线程去处理该任务，而此时假如之前有一些线程完成了任务，现在又有新任务提交，那么将不会创建新线程去处理，而是复用空闲的线程去处理新任务。那么此时有人有疑问了，那这样来说该线程池的线程岂不是会越集越多？其实并不会，因为线程池中的线程都有一个“保持活动时间”的参数，通过配置它，如果线程池中的空闲线程的空闲时间超过该“保存活动时间”则立刻停止该线程，而该线程池默认的“保持活动时间”为60s。

3. newSingleThreadExecutor() ： 
该方法返回一个只有一个线程的线程池，即每次只能执行一个线程任务，多余的任务会保存到一个任务队列中，等待这一个线程空闲，当这个线程空闲了再按FIFO方式顺序执行任务队列中的任务。

4. newScheduledThreadPool() ： 
该方法返回一个可以控制线程池内线程定时或周期性执行某任务的线程池。

5. newSingleThreadScheduledExecutor() ： 
该方法返回一个可以控制线程池内线程定时或周期性执行某任务的线程池。只不过和上面的区别是该线程池大小为1，而上面的可以指定线程池的大小。

使用时，官方建议我们用Executors来创建一个线程池，然后执行ExecutorService的execute(Runnable command) 方法来执行。其中Runnable对象的调度是有池的类型来决定的。完整的流程是这样的：

```
 ExecutorService singleThreadPool = Executors.newSingleThreadExecutor();
       singleThreadPool.execute(new Runnable() {
                @Override
                public void run() {
                    //todo
            });
        }
```
这样我们就创建了一个singelThreadPool，然后使用execute去执行相关的方法。我们可以用Executors创建不同种类的线程池，然后来管理线程进行异步处理。
