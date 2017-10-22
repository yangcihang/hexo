---
title: Android消息机制
date: 2017-10-21 23:13:29
tags: Android
categories: Android
---
## Handler简介
我们都知道Handler是Android消息机制的上层接口，用Handler可以将一个任务切换到Handler所在的线程去执行。Handler最最常见的操作就是在子线程中来更新UI线程的视图。常用的方法有：<!--more-->

* void handleMessage(Message msg)处理消息的方法,通常是用于被重写
* sendEmptyMessage(int what)sendEmptyMessageDelayed(int what,long delayMillis) 发送空消息
* sendMessage(Message msg):sendMessageDelayed(Message msg):发送消息
* final boolean hasMessage(int what):检查消息队列中是否包含what属性为指定值的消息 如果是参数为(int what,Object object):除了判断what属性,还需要判断Object属性是否为指定对象的消息
* post(Runnable r),postDelayed(Runnable r, long delayMillis)Runnable的执行是在Handler对象所在的线程，因此如果Handler在主线程的话，还是别用这个方法来执行耗时操作了，不然就ANR。

## 消息机制
Android消息机制其实上是Handler的运行机制及Handler附带的MessageQueue和Looper的工作过程。这里先说明一点，安卓UI控件是线程不安全的，因此不允许在子线程中对UI控件进行修改。那为啥设计的时候不加锁呢？可能是加锁会降低访问效率和阻塞线程的原因。

Handler创建的时候会采用当前线程的Looper来构建内部的消息循环系统，如果当前线程没有Looper，就会报错。主线程默认会创建Looper。当Handler创建完毕后，这时候内部的Looper以及MessageQueue就可以和Handler一起工作了。然后通过Handler的post方法将一个Runnable传递到Looper中去处理，或者通过Handler的send方法发送消息，这个消息同样也会在Looper中去处理。其实post方法最终也是通过send方法来完成的。当Handler的send方法调用时，它会调用MessageQueue的enqueueMessage方法将这个消息放入消息队列，然后Looper发现有新的消息的时候，就会处理这个消息，最终消息的Runnable或者Handler的handleMessage方法就会被调用。在介绍这三个机制之前先说说java.lang包下面的ThreadLocal类吧。

### ThreadLocal
什么是ThreadLocal呢？ThreadLocal是java.lang包下面的一个线程内部的数据存储类，通过它可以在指定线程存储指定数据，并且只有在指定线程中获得存储的数据。一般来说，当某些数据是以线程为作用域并且不同线程具有不同的数据副本时，就可以考虑ThreadLocal。举个栗子：监听器的传递，如果一个线程中做了大量工作而且需要一个监听器来贯穿整个线程的执行过程，就可以使用ThreadLocal，在线程内部使用ThreadLocal的get方法来获取到监听器。举个ThreadLocal简单的例子，在主线程中创建一个ThreadLocal的实例，类型选择为int型的，并且在主线程中用set方法设置为1，然后在子线程1中set为2，子线程2中不进行任何操作。如果将三个线程调用get方法，则在主线程中会显示1，在子线程1中显示2，在子线程2中则会显示null.虽然在不同线程中访问的是同一个ThreadLocal的对象，但是他们的值是不同的。为什么会这样呢？来看看get方法的源码：

```  
  public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```    
我们可以看到，每个Thread都会对应一个ThreadLocalMap，ThreadLocalMap其实是ThreadLocal的一个私有内部类，在ThreadLocal里面set的时候会调用map的set方法将数据存储在一个叫Entry的数组里面。为啥要先说这个ThreadLocal呢？因为等会介绍的Looper的作用域就是线程并且不同线程具有不同的Looper。

### MessageQueue
MessageQueue顾名思义，就是存放Message的队列，然而它内部的数据结构并不是队列，而是用单链表存储的消息列表，因为链表的插入删除很方便。主要有俩方法：enqueueMessage和next。其中enqueueMessage会在Handler每次发送Message时候调用，将Message加入链表之中。源码也是将Message插入链表的意思。然后主要是里面的next方法：

```
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
                ......
```
很明显这个next方法是个for无限循环的方法。如果没有消息的话会一直阻塞在这，当有消息到来时会将其从单链表中移除。这个next方法会在接下来的Looper的某一方法中调用。

### Looper
Looper是一个不停地在MessageQueue中查看是否有新消息的一个东西。如果没有新消息就会阻塞在那里。**在它的构造方法里面会创建一个本线程的MessageQueue**，然后将当前线程的对象保存起来。一般如果要在Android的非UI线程中进行跨线程的操作时，就需要创建一个Looper（因为UI线程的Looper比较特殊，会在一开始就创建）。Looper提供了quit和quitSafely方法，区别是前者是立即退出，后者是处理完MessageQueue中的所有Message后退出。当Looper退出后，Handler的sendMessage方法就会返回false。先看看Looper的prepare的方法：

```
 public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
可以看到，prepare的方法会调用ThreadLocal的set方法set一个Looper的一个对象。这样就保证了Looper在不同线程中有不同的Looper存在。然后咱们看看Looper中最核心的方法：loop

```
  public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
可以看到，loop中有一个for的无限循环，当MessageQueue的next方法返回为空时，退出这个循环。前面刚说过了这个next方法是一个阻塞的方法，当MessageQueue被标记为退出状态时，这个next就会返回一个null。其实Looper的quit方法就是调用MessageQueue方法中的quit来实现的。接下来当获取到消息的时候，会执行msg.target.dispatchMessage方法。这个target就是发送消息的Handler对象，这个dispatchMessage方法接下来会说。

### Handler
先说明一点，Handler中的post方法最终都会执行到sendMessageAtTime这个方法中去，然后将Message加入MessageQueue中。然后Looper会不停的循环这个MessageQueue，当取出消息后就会执行到Handler的dispatchMessage的方法中去。

```
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

```
可以看到这个方法也是很简单就理解的。其中这个callback是一个Runnable对象，实际上就是Handler的post方法中的Runnable。然后检查这个Callback是否为空。这个Callback其实是一个接口，在handler初始化的时候传入的。这个接口可以实现处理Message的一些方法，其实和Handler的handlerMessage方法一样，用这个接口的话可以避免写Handler子类去做一些处理。最后调用的这个方法应该很熟悉吧？没错，平常创建Handler时候重写的那个方法，其实它是在dispatchMessage方法中进行回调的。

参考文献：Android开发艺术探究