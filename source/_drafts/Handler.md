---
title: Handler
tags:
categories:
---



`Handler`经常被用于**异步消息通知**与**延时任务处理**。

在子线程中处理数据后，此时需要在UI线程回显数据，在主线程创建`Handler`对象后，子线程通过`Handler.sendMessage()`方法发送到handler的处理队列中，在`handleMessage`方法中做相应的处理。

延时任务通过`postDelayed`方法，一定时间后执行传入的`Runnable`对象。



### 源码分析

`Handler`中的关键类为`Looper`,其提供了一个`MessageQueue`消息队列，无论是将要发送的`message`还是执行的`runnable`都会被封装为`Meassage`类压入消息队列中，`Looper`中通过`loop`方法中的死循环，根据压入的时间或延时时间，顺序分发队列中的消息。



#### 构建Handler

```java
public Handler(Callback callback, boolean async) {

    mLooper = Looper.myLooper(); // 获取当前线程中的Looper对象
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue; // 获取到Looper中的队列引用
    mCallback = callback;
    mAsynchronous = async;
}
```

#### 创建Looper和MessageQueue

`Looper`对象创建后会保存到当前线程的`ThreadLocal`中，而主线程`ActivityThread`在启动时，便创建了本线程的`Looper`对象。

``` java ActivityThread.java
public static void main(String[] args) {
    Looper.prepareMainLooper(); // 创建主线程Looper
    ...
    ActivityThread thread = new ActivityThread();
    ...
    Looper.loop(); // 开始死循环
}
```



```java Looper.java
public static void prepareMainLooper() {
    prepare(false); // 创建Looper
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();// 将当前的looper对象记录为主线程的Looper
    }
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed)); // 将新创建Looper设置到ThreadLocal中
}
```



而`Looper`中的`MessageQueue`，则在`Looper`的构造方法中一并创建。

```java Looper.java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed); // 创建消息队列
    mThread = Thread.currentThread();
}
```

通过上述步骤，主线程的`Handler`即可工作起来，而如果想要在子线程中使用`Handler`，则需要手动创建Looper，并开启循环。或者直接使用主线程的`looper`对象

```java
new Thread(){   
   @Override    
   public void run() {        
       Looper.prepare();  //创建当前线程的Looper
       Handler handler = new Handler() {           
           @Override          
           public void handleMessage(Message msg) {                
               // ...          
           }        
       };        
       handler.sendEmptyMessage(0);   
       Looper.loop();  //looper开始处理消息。
   }
}.start();
```



#### MessageQueue压入消息

如果是通过`post()`方法传入了一个`Runnale`，在`handler`中会通过`getPostMessage()`方法封装为`message`对象。

```java Handler.java
public final boolean postDelayed(Runnable r, long delayMillis){
    return sendMessageDelayed(getPostMessage(r), delayMillis); // 封装为Message对象
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r; // 将Runnable设置为Message的callback参数
    return m;
}
```

而通过`sendMessage`发出的消息，最终与`postDelayed`一样，都会调用到`sendMessageDelayed()`，最终调用到`enqueueMessage()`方法。

```java Handler.java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this; // 将Handler自身的引用设置到msg上，当msg的处理时间到后，会分发到该Handler中处理
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);// 压入消息队列中
}
```

最终通过调用`queue.enqueueMessage()`将消息加入到队列中。



在`MessageQueue`中，消息存放是一个链表，`mMessages`字段为链表头，方法中传入的`when`字段为该message预期执行的时间，通过遍历列表，通过判断链表中已有的消息的时间，使插入新消息后，链表中的消息时间顺序由小到大。

```java MessageQueue
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        // ...

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;// 头指针
        boolean needWake;
        if (p == null || when == 0 || when < p.when) { 
            // 如果头指针为空，或第一个消息的执行时间都晚于新消息，则新消息成为头指针
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; 
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) { // 逐步向后遍历
                prev = p;
                p = p.next;
                if (p == null || when < p.when) { // 找到尾节点或执行时间晚于新消息的节点，跳出循环
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // 加入到列表中
            prev.next = msg;
            // 如果之前没有任务处理，进入了阻塞状态，则压入消息后需要唤醒线程
            if (needWake) {
                nativeWake(mPtr);
            }
        }

    }
    return true;
}
```

#### loop处理消息

当Looper创建后，需要调用loop方法进入死循环模式，当`MessageQueue`中出现了新消息，则唤醒线程，推出一个消息，交由Looper处理。

```java Looper.java
public static void loop() {
        final Looper me = myLooper();
        if (me == null) { // 检测线程中是否存在Looper对象
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }

        final MessageQueue queue = me.mQueue;

        for (;;) { // 进入死循环
            Message msg = queue.next(); // might block，当存在消息后会唤醒线程，推出Message
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            try {
                msg.target.dispatchMessage(msg); // 交由send该Message的Handler处理
             
            } catch (Exception exception) {
                throw exception;
            } 

            msg.recycleUnchecked(); // 对 Message 进行了回收
        }
    }
```

取出`Message`后，交由`Message`中的`target`对象处理，而这个`target`对象则就是在`Handler.enqueueMessage()`的方法中赋值的。

Handler中处理这个Message。

```java
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) { // 如果是Runnable
            handleCallback(msg); //  则直接run。 (message.callback.run();)
        } else {
            // 如果是正常的消息，则调用handleMessage方法处理。
            if (mCallback != null) {
                // 回调到构造Handler时传入的callback中
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            // 默认为空实现，如果有子类实现，则可以重写该方法处理消息。
            handleMessage(msg);
        }
    }
```



#### MessageQueue推出Message机制











https://mp.weixin.qq.com/s/Ylc5mPwMzWoK2CIthZy0Vw