## 1. 为什么会出现Handler？

在Android开发中：

* 为了保证Android的UI操作是**线程安全**的，Android规定**只允许UI线程**修改界面上的UI组件；
* 但在实际开发中，必然会遇到多个线程并发操作UI组件，这又将导致UI操作的线程不安全。

Handler消息传递机制就是为了同时满足：

* 保证线程安全
* 使多个线程并发操作UI组件

## 2. 相关概念

**主线程（UI线程）**

* 定义：当程序第一次启动时，Android会同时启动一条主线程（Main Thread）
* 作用： 主线程主要负责处理与UI相关的事件，所以主线程又叫UI线程

**Message**

* 定义：消息，理解为线程间通讯的数据单元（Handler接受和处理的消息对象）；Message包含了**next**，next是Message的实例；由此可见，Message是一个**单链表**。Message还包括了**target**成员，target是Handler实例。此外，它还包括了arg1，arg2，what，obj等参数，它们都是用于记录消息的相关内容

**MessageQueue**

* 定义：消息队列；包含了**mMessages**成员；mMessages是消息Message的实例。MessageQueue提供了**`next()`**方法来获取消息队列下一个消息。
* 作用：用来存放通过Handler发过来的消息，按照**FIFO**执行

**Looper**

* 定义：消息泵，扮演Message queue和handler之间桥梁的角色；包括了mQueue成员变量；mQueue是消息队列MessageQueue的实例。还
* 作用：通过`loop()`方法进入到消息循环中取出Message，将取出的message交付给对应的Handler

**Handler**

* 定义：消息句柄类
* 作用：发送Message，处理Message

## 3. Handler工作流程解释

异步通信传递机制步骤主要包括异步通信的准备、消息发送、消息循环和消息处理

1. 异步通信的准备

   包括Looper对象的创建&实例化、MessageQueue的创建和Handler实例化

2. 消息发送

   Handler将消息发送到消息队列中

3. 消息循环

   Looper执行Looper.loop()进入到消息循环，在这个循环过程中，不断从该MessageQueue取出消息，并将取出的消息派发给创建该消息的Handler

4. 消息处理

   调用Handler的dispatchMessage(msg)方法，即回调handleMessage(msg)处理消息

## 4. 根据源码详细讲解

### Looper.class

```java
public final class Looper{
  
  // sThreadLocal.get() will return null unless you've called prepare().
  static final ThreadLocal<Loopre> sThreadLocal = new ThreadLocal<Looper>();
  
  private Looper(boolean quitAllowed){
  	mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
  }
  
  public static void prepare(){
  	prepare(true); //此处的传参为ture
  }
  
  public static void prepare(boolean quitAllowed){
  	if(sThreadLocal.get() != null){
      throw new RuntimeException("Only one Looper may be created per thread");
	}
    sThreadLocal.set(new Looper(quitAllowed));
  }
  
  public static void prepareMainLooper(){
    prepare(false);//此处的传参为false
    synchronized (Looper.class){
  		if(sMainLooper != null){
  			throw new IllegalStateException("The main Looper has already been prepared.")
            sMainLooper = myLooper();
		}
	}
  }
  
  public static void loop(){
    final Looper me = myLooper();
    if (me == null){
  		throw new RuntimeException("No Looper;Looper.prepare() wasn't called on this thread.");
        final MessageQueue queue = me.mQueue;
        ...
        for(;;){
  			Message msg = queue.next(); // might block
            if (msg == null){
  			  // No message indicates that the message is quitting.	
              return;
			}
            ...
            msg.target.dispatchMessage(msg); //消息的分发;mag.target下面将会看到
            ...
            msg.recycleUnchecked();
		}
	}
  }
  
  public static Looper myLooper(){
     return sThreadLocal.get();
  }
  
}
```
### Handler.class

```java
public class Handler{
  
  public Handler(){
     this(null,false) ;
  }
  
  public Handler(Callback callback,boolean async){
     if(FIND_POTENTIAL_LEAKS){
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() ||     klass.isLocalClass()) &&(klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " + klass.getCanonicalName());
        }
     }
    // 通过Looper.myLooper()获取了当前线程保存的Looper实例，如果线程没有Looper实例那么会抛出异常
    // 这说明在一个没有创建Looper的线程中是无法创建一个handler对象的
    // 得出在一个线程中创建Handler时首先需要创建Looper，并且开启消息循环才能够使用这个Handler
     mLooper = Looper.myLooper();
     if(mLooper == null){
       throw new RuntimeException("Can't create handler inside thread that has not called Looper.prepare()");
     }
    // 获取了这个Looper实例中保存的MessageQueue
    // 也让handler的实例和Looper实例中的MQ关联上了
     mQueue = mLooper.mQueue;
     mCallback = callback;
     mAsynchronous = async;
  }
  
}
```

Handler向MessageQueue发消息，方式可以分为两种**post**和**send**。

**最常用的`sendMessage()`方法** 

```java
public final boolean sendMessage(Message msg)
{
   return sendMessageDelayed(msg, 0);
}
```

```java
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}
```

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
   if (delayMillis < 0) {
       delayMillis = 0;
   }
   return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

```java
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
```

归根结底还是**调用的`sendMessageAtTime(Message msg, long uptimeMillis) `**方法，在此方法内部获取了MessageQueue然后调用了`enqueueMessage()`方法。

```java
private boolean enqueueMessage(MessageQueue queue,Message msg,long uptimeMillis){
  //如果还记得Looper的loop方法会取出每个msg然后交msg,target.dispatchMessage(msg)去处理消息
  //也就是把当前的handler作为msg的target属性
  msg.targer = this;
  if(mAsynchronous){
     msg.setAsynchronous(true);
  }
  return queue.enqueueMessage(msg.uptimeMillis);
}
```

最终调用queue的enqueueMessage方法，也就是说handler发出的消息，最终会保存到消息队列中。

之前已经清除知道Looper会调用prepare和loop方法，在**当前执行的线程中**保存一个**Looper**实例，这个实例会保存一个MessageQueue对象，然后当前线程进入一个无限循环中去，不断从MessageQueue中读取Handler发来的消息。然后再回调创建这个消息的handler中的dispatchMessage方法。

```java
public void dispatchMessage(Message msg){
  if(msg.callback != null){
     handlerCallback(msg);
  }else{
     if(mCallback != null ){
  		if(mCallback.handleMessage(msg)){
           return;
		}
     }
    handlerMessage(msg);
  }
}
```

可以看到最后调用了`handlerMessage`方法，进入这个方法中

```java
/**
* SubClasses must implement this to receive messages.
*/
public void handleMessage(Message msg){
  
}
```

可以看到这是一个空方法，为什么呢？因为消息的最终回调是由我们自己控制的，在创建handler的时候都是复写handleMessage方法，然后根据msg.what进行消息处理。

例如：

```java
private Handler mHandler = new Handler()
{
    public void handleMessage(android.os.Message msg)
    {
		switch (msg.what)
		{
		 case value:
				
			break;

		  default:
				break;
		}
	};
};
```

**Handler post** 

```java
mHandler.post(new Runnable()
{
	@Override
	public void run()
	{
		Log.e("TAG", Thread.currentThread().getName());
		mTxt.setText("yoxi");
	}
});
```

`run()`方法中可以写更新UI的代码，其实这个Runnable并**没有创建什么线程**，而是**发送了一条消息**

```java
 public final boolean post(Runnable r)
 {
    return  sendMessageDelayed(getPostMessage(r), 0);
 }
```

```java
 private static Message getPostMessage(Runnable r) {
     Message m = Message.obtain();
     m.callback = r;
     return m;
 }
```

可以看到，在`getPostMessage()`中，得到了一个Message对象，然后将我们创建的**Runnable对象**作为**callback**属性，赋值给了Message。

`sendMessageDelayed()`方法调用的还是`sendMessageAtTime()`，然后调用了`enqueueMessage()`方法，给msg.target赋值为handler，最终加入MessageQueue。

最终在`dispatchMessage()`方法判断执行哪个？

```java
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

```java
private static void handleCallback(Message message) {
   message.callback.run();
}
```

第2行，如果不为null，则执行callback回调，也就是我们的Runnable对象。

### 流程总结

1. 首先Looper.prepare()在本线程中保存一个Looper实例，然后该实例中保存一个MessageQueue对象；因为Looper.prepare()在一个线程中**只能调用一次**，所以MessageQueue在一个线程中只会存在一个。
2. **Looper.loop()**会让当前线程进入一个**无限循环**，不断从MessageQueue的实例中读取消息，然后回调**msg.target.dispatchMessage(msg)**方法
3. Handler的构造方法，会首先得到当前线程中保存的Looper实例，进而与Looper实例中的MessageQueue相关联
4. Handler的sendMessage方法，会给msg的target赋值为**handler自身**，然后加入到MessageQueue 
5. 在构造Handler实例时，会**重写handleMessage()**方法，也就是**msg.target.dispatchMessage(msg)**最终调用的方法 

### 结束MessageQueue消息队列阻塞死循环源码分析

Looper类的**`quit`**方法源码：

```java
public void quit(){
  mQueue.quit(false);
}
```

可以看到是调用了MessageQueue的**`quit`**

```java
void quit(boolean safe){
  if(!mQuitAllowed){
     throw new IllegalStateException("Main thread not allowed to quit.");
  }
  
  synchronized (this){
     if(mQuitting){
        return;
     }
     mQuitting = true;
     if(safe){
        removeAllFutureMessagesLocked();
     } else{
        removeAllMessagesLocked();
     }
    
     // We can assume mPtr != 0 because mQuitting was previously false
     nativeWake(mPtr);
  }
}
```

方法中通过判断标记**mQuitAllowed**来决定该消息队列是否可以退出，然而当**mQuitAllowed**为**false**时抛出的异常是**"Main thread not allowed to quit"**, Main Thread，所以可以说明Main Thread关联的Looper----对应的MessageQueue消息队列是**不能通过该方法退出**的。

而这个**mQuitAllowed**在哪设置的？

是**MessageQueue**构造函数传递参数传入的，而**MessageQueue**对象的实例化是在**Looper**的构造函数实现的，所以不难发现**mQuitAllowed**参数实质是从**Looper**的**构造函数**传入的。**Handler**实例化分析时说过，**Looper实例化**是在**Looper**的静态方法**prepare(boolean quitAllowed)**中处理的，也就是说**mQuitAllowed**是由**Looper.prepare(boolean quitAllowed)**参数传入的。追根到底说明**mQuitAllowed**就是**Looper.prepare**的参数，默认调用的**Looper.prepare()**其中对**mQuitAllowed**设置为**true**，所以可以通过`quit()`方法退出，而主线程ActivityThread的main中使用的是`Looper.prepareMainLooper()`;这个方法里对mQuitAllowed设置为**false** ，所以才会上面的"Main thread not allowed to quit"

继续看`quit()`方法，可以发现实质就是对**mQuitting**标记置位，这个**mQuitting**标记在MessageQueue的阻塞等待`next()`中做了判断条件，所以可以通过`quit()`方法退出整个当前线程的loop循环

## 参考

[Android 异步消息处理机制 让你深入理解 Looper、Handler、Message三者关系](http://blog.csdn.net/lmj623565791/article/details/38377229)

[Android异步消息处理机制完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/9991569)

[Android中关于Handler的若干思考](http://www.cnblogs.com/smyhvae/p/4799730.html)

[Android异步消息处理机制详解及源码分析](http://blog.csdn.net/yanbober/article/details/45936145)

[Android消息机制架构和源码解析](http://wangkuiwu.github.io/2014/08/26/MessageQueue/)

[Android开发：Handler异步通信机制全面解析(包含Looper、Message Queue)](http://www.jianshu.com/p/9fe944ee02f7)