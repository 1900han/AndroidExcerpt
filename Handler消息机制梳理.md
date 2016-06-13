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

1. **Looper.class**

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

2. **Handler.class**

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

   ​