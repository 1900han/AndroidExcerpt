拖了几个月了,还是写吧.虽然写的和流水账一样

### 使用

这里主要记录如何使用3.0的新特性-索引

#### 索引先决条件

只有**订阅者**和**事件类**是**public**,**@Subscriber**方法才能被索引.并且,由于Java注解处理器本身的技术限制,@Subscribe注解在**匿名内部类中不被识别**.

当EventBus不能使用索引,将会在运行时自动回到反射.

#### 如何生成索引

为了能生成索引,需要zaiandroid -apt gradle插件的编译环境添加EventBus annotation processer .

还需设置一个参数eventBusIndex到指定完全限定的想要生成的索引类.因此,添加下面的代码到你的Gradle 编译脚本中

```groovy
buildscript {
    dependencies {
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}
 
apply plugin: 'com.neenbedankt.android-apt'
 
dependencies {
    compile 'org.greenrobot:eventbus:3.0.0'
    apt 'org.greenrobot:eventbus-annotation-processor:3.0.1'
}
 
apt {
    arguments {
        eventBusIndex "com.example.myapp.MyEventBusIndex"
    }
}
```

当下次编译项目,"eventBusIndex"这个类会生成.然后当设置EventBus像这样

```java
EventBus eventBus = EventBus.builder().addIndex(new MyEventBusIndex()).build();
```

或者,你想在你的APP使用默认的EventBus实例

```java
EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
// Now the default instance uses the given index. Use it like this:
EventBus eventBus = EventBus.getDefault();
```

#### 类库索引

你可以使用相同的代码在你的类库(不是最终的应用),这样,会有多个索引类,在设置EventBus你可以调用多个,例如

```java
EventBus eventBus = EventBus.builder()
    .addIndex(new MyEventBusAppIndex())
    .addIndex(new MyEventBusLibIndex()).build();
```

### 源码

从四个方面入手**创建,注册,发送事件,注销**

#### 创建

```java
    /**
     * Convenience singleton for apps using a process-wide EventBus instance.
     */
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```

使用了**单例模式**,目的是为了保证`getDefault()`得到的都是同一个实例。如果不存在实例,就调用了`EventBus`的构造方法:

```java
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
```

![](http://o75vlu0to.bkt.clouddn.com/EventBus.EventBus%28%29.png)

![](http://o75vlu0to.bkt.clouddn.com/EventBus.EventBus%28EventBuilder%20builder%29.png)

这里通过**EventBuilder**对象初始化**EventBus**.

#### 注册

```java
EventBus mEventBus = EventBus.builder().addIndex(new MyEventBusIndex()).build();
mEventBus.register(this);
```

注意这里是使用了索引.读源码的顺序和正常使用有区别

![](http://o75vlu0to.bkt.clouddn.com/EventBus.register%28%29.png)

这里主要是这句代码

![](http://o75vlu0to.bkt.clouddn.com/findSubscriberMethods.png)

点进去,看详细代码

![](http://o75vlu0to.bkt.clouddn.com/SubscriberMethodFinder.findSubscriberMethods%28%29.png)

看到有两种查找订阅方法的方式

现在使用的是索引,所以看`findUsingInfo()`

![](http://o75vlu0to.bkt.clouddn.com/SubscriberMethodFinder.findUsingInfo%28%29.png)

里面的FindState,值得认真看一下

![](http://o75vlu0to.bkt.clouddn.com/FindState-1.png)

![](http://o75vlu0to.bkt.clouddn.com/FindState-2.png)

`checkAdd()`分为两个层级的check,第一层级只判断event type,这种方式速度快;第二层是多方面判断.anyMethodByEventType是一个HashMap,`hashMap.put()`方法返回的是之前的value,如果之前没有value的话返回的是null,通常一个订阅者(包括继承关系)不会有多个系统方法接收同一事件,但是可能会出现子类订阅这个事件的同时父类也订阅了此事件的情况,那么`checkAddWithMethodSignature()`就排上了用场.

**这里主要思想就是不要出现一个订阅者(主要是继承关系的订阅者)有多个相同方法订阅的是同一事件.**

最终返回**List<SubscriberMethod>**.

看下SubscriberMethod类

![](http://o75vlu0to.bkt.clouddn.com/SubscriberMethod.png)

封装了EventBus所需要的全部信息

#### 利用索引的查找订阅方法看完了,看另外一种方法,使用反射

![](http://o75vlu0to.bkt.clouddn.com/SubscriberMethodFinder.findUsingReflection%28%29.png)

![](http://o75vlu0to.bkt.clouddn.com/SubscriberMethodFinder.findUsingReflectionInSingleClass%28%29.png)

#### 总结下subscriberClass过程

1. 从复用池中拿到或者new一个FindState
2. 将subscriberClass复制给findState
   1. 进入循环,判断当前clazz不为null
   2. 再判断findState.subscriberInfo是否为null
      1. 不为null,说明使用索引
      2. 为null,进入findUsingReflectionSingleClass()
   3. 将clazz变为clazz的父类,再次进行循环的判断
3. 返回所有的SubscriberMethod

#### 订阅

![](http://o75vlu0to.bkt.clouddn.com/EventBus.subscribe%28%29-1.png)

![](http://o75vlu0to.bkt.clouddn.com/EventBus.subscribe%28%29-2.png)

#### 发布事件

```java
EventBus.getDefault().post("str");
```

![](http://o75vlu0to.bkt.clouddn.com/EventBus.post%28%29.png)

![](http://o75vlu0to.bkt.clouddn.com/EventBus.postSingleEventForEventType%28%29.png)

![](http://o75vlu0to.bkt.clouddn.com/EventBus.postToSubscription%28%29.png)

#### 注销订阅

![](http://o75vlu0to.bkt.clouddn.com/EventBus.unregister%28%29.png)

![](http://o75vlu0to.bkt.clouddn.com/EventBus.unsubscribeByEventType%28%29.png)

#### EventBus中的线程模型

EventBus中共有四种线程模型:**PostThread,MainThread,BackgroundThread,Async**

##### PostThread

默认的线程模型,事件发布和接收在同一个线程,**适合用于完成耗时非常短的操作,以免阻塞UI**.

##### MainThread

订阅者的事件处理方法在主线程被调用,**适合处理开销小的事件**.以免阻塞主线程.

##### BackgroundThread

订阅者的事件处理在后台被调用.若事件发送线程位于主线程,EventBus将使用一个后台线程来逐个发送事件.

##### Async

事件处理位于一个单独的线程,该线程往往既不是发送事件的线程,也不是主线程.使用这个模型**适用于事件发送无需等待事件处理后才返回,适合处理耗时操作**,如请求网络.EventBus内部使用线程池来管理多个线程.

下面从源码看这几种线程模型的调用

主要入口是EventBus中的`postToSubscription(Subscription subscription, Object event, boolean isMainThread)`方法

![](http://o75vlu0to.bkt.clouddn.com/EventBus.postToSubscription%28%29.png)

通过解析线程模型,如果是MainThread线程模型,并且当前线程就是主线程,那么可以直接在当前主线程通过反射调用订阅者的事件处理方法.

如果当前线程不是主线程,就借助Handler,将此事件发送到主线程处理.

MainThreadPoster是EventBus的一个属性,类型为HandlerPoster.

在EventBus的构造函数中是这样设置

```java
mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
```

传入的**第二个参数**是**主线程的Looper**,**第三个参数**表示**主线程事件处理最大时间为10ms**,超时将重新调度事件.



### 参考

[老司机教你 “飙” EventBus 3](http://bugly.qq.com/bbs/forum.php?mod=viewthread&tid=938&fromuid=1147)

[EventBus3.0源码解析](http://yydcdut.com/2016/03/07/eventbus3-code-analyse/#)

[EventBus源码分析（一）：入口函数提纲挈领（2.4版本）](http://blog.csdn.net/wangshihui512/article/details/51802172)

[EventBus 3.0 源代码分析](http://skykai521.github.io/2016/02/20/EventBus-3-0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)