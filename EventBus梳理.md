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

从三个方面入手**注册**,**发送事件**,**注销**

#### 注册

```java
EventBus mEventBus = EventBus.builder().addIndex(new MyEventBusIndex()).build();
mEventBus.register(this);
```



























