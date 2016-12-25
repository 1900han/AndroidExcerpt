### RxJava到底是什么?
一个词:**异步**
RxJava 在 GitHub 主页上的自我介绍是 "a library for composing asynchronous and event-based programs using observable sequences for the Java VM"（一个在 Java VM 上使用可观测的序列来组成异步的、基于事件的程序的库）
其实， RxJava 的本质可以压缩为异步这一个词。说到根上，它就是一个实现异步操作的库，而别的定语都是基于这之上的。
### RxJava好在哪?
换句话说,同样是做异步,为什么人们用它,而不用现成的AsyncTask/Handler/XXX/...?
一个词:**简洁**
### API介绍和原理简析
#### RxJava的观察者模式
**四个基本概念**
* Observable (被观察者)
* Observer (观察者)
* subscribe (订阅)
* 事件
    * onNext()
    * onCompleted()
            事件队列完结.RxJava不仅把每个事件单独处理,还会把它们看作是一个队列.RxJava规定,当不会再有新的onNext()发出时,需要触发onCompleted方法作为标志.
    * onError()
            事件队列异常.在事件处理过程中出异常时,onError()会被触发,同时队列自动终止,不允许再有事件发出.
    * 在一个正确运行的事件序列中,onCompleted()和onError()有且只有一个,并且是事件序列中的最后一个.需要注意的是,onCompleted()和onError()是互斥的,即在队列中调用了其中一个,就不应该再调用另一个.
#### 基本实现
基于以上的概念,RxJava实现主要有三点:
**1.创建Observer**
    Observer即观察者,它决定事件触发的时候将有怎样的行为.RxJava中的Observer接口实现:
``` Java
Observer<String> observer = new Observer<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};
```
除了Observer接口外,RxJava还内置了一个实现了Observer   的抽象类:Subscriber.Subscriber对Observer接口做了一些扩展,但他们的基本使用方式是一样的:
```Java
Subscriber<String> subscriber = new Subscriber<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};
```
不仅使用方式一样,实质上,在RxJava的subscriber过程中,Observer也总是会先被转成一个Subscriber再使用.如果只是使用基本功能,选择谁都是一样的.它们的区别对使用者来说主要又两点:
    1. `onStart()`:这是Subscriber增加的方法.它会在subscribe刚开始,事件还未发送之前被调用,可以用于做一些准备工作,例如**数据**的**清零**或**重置**.这是一个可选方法,默认情况下它的实现为空.需要注意的是,如果对**准备工作的线程有要求**(例如弹出一个显示进度的对话框,这必须在主线程执行),`onStart()`就**不适用**了,因为它总是在subscribe所发生的线程被调用,而不能指定线程.要在指定的线程做准备工作,可以使用`doSubscribe()`方法,具体可以在后面的文章看到.
    2. `unsubscribe()`:这是Subscriber实现另一个接口Subscription的方法,用于取消订阅.在这个方法调用后,Subscriber将不再接收事件.一般在这个方法调用前,可以使用`isUnsubscribed()`判断以下状态.`unsubscribe()`这个方法很重要,因为在subscribe之后,Observable会持有Subscriber的引用,这个引用如果不能及时被释放,将有内存泄漏的风险.所以最好保持一个原则:要在不再使用的时候尽快在合适的地方(例如`OnPause()` `onStop()`等方法中)调用`unsubscribe()`来解除引用关系,以避免内存泄漏的发生.
**2.创建Observable**
    Observable即被观察者,它决定什么时候触发事件以及触发怎样的事件.RxJava使用`create()`方法来创建一个Observable,并为它定义事件触发规则.
```Java
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello");
        subscriber.onNext("Hi");
        subscriber.onNext("Aloha");
        subscriber.onCompleted();
    }
});
```
可以看到,这里传入了一个OnSubScribe对象作为参数.OnSubscribe会被存储在返回的Observable对象中,它的作用相当于一个计划表,当Observable被订阅的时候,OnSubscribe的`call()`方法会被自动调用,事件序列就会被依照设定依次触发(对于上面的代码,就是观察者Subscriber将会被调用三次onNext()和一次onCompleted().这样,由被观察者调用了观察者的回调方法,就实现了由被观察者向观察者的事件传递,即观察者模式.
`create()`方法是RxJava最基本的创造事件序列的方法.基于这个方法,RxJava还提供了一些方法用来快捷创建事件队列,例如
* `just(T...)`将传入的参数依次发送出来
```Java
Observable observable = Observable.just("Hello", "Hi", "Aloha");
// 将会依次调用：
// onNext("Hello");
// onNext("Hi");
// onNext("Aloha");
// onCompleted();
```
* `from(T[])/from(Iterable<? extends T>)`将传入的数组或Iterable拆分成具体对象后,依次发送出来.
```Java
String[] words = {"Hello", "Hi", "Aloha"};
Observable observable = Observable.from(words);
// 将会依次调用：
// onNext("Hello");
// onNext("Hi");
// onNext("Aloha");
// onCompleted();
```
上面`just(T...)`的例子和`from(T[])`的例子,都和之前的create(OnSubscribe)的例子是等价的.
**3.Subscribe (订阅)**
创建了Observable和Observer之后,再用`subscribe()`方法将它们联接起来,整条链子就可以工作了.代码形式很简单:
```Java
observable.subscrib(observer);
//或者
observable.subscribe(subscriber);
```
> 有人可能会注意到, `subscribe()`这个方法有点怪:它看起来是[obsevable订阅了observer/subscriber]而不是[observer/subscriber订阅了observable],这看起就像[杂志订阅了读者]一样颠倒了对象关系.这让人读起来有点别扭,不过如果把API设计成observer.subscriber(observable)/subscriber.subscribe(observable),虽然更加符合思维逻辑,但对流式API的设计就造成影响了,比较起来明显是得不偿失的.
`Observable.subscribe(Subscriber)`的内部实现是这样的(仅核心代码)
```Java
// 注意：这不是 subscribe() 的源码，而是将源码中与性能、兼容性、扩展性有关的代码剔除后的核心代码。
// 如果需要看源码，可以去 RxJava 的 GitHub 仓库下载。
public Subscription subscribe(Subscriber subscriber) {
    subscriber.onStart();
    onSubscribe.call(subscriber);
    return subscriber;
}
```
可以看到,`subscriber()`做了3件事:
1. 调用Subscriber.onStart().这个方法在前面已经介绍过,是一个可选的准备方法.
2. 调用`Observable`中的OnSubscribe.call(Subscriber).在这里,事件发送的逻辑开始运行.从这也可以看出,在RxJava中,`Observable`并不是在创建的时候就立即开始发送事件,而是在它被订阅的时候,即当`subscribe()`方法执行的时候.
3. 将传入的`Subscriber`作为Subscription返回这是为了方便`unsubscribe()`.
整个过程中对象间的关系如下图:
![](http://ww3.sinaimg.cn/mw1024/52eb2279jw1f2rx4ay0hrg20ig08wk4q.gif)
除了`subscribe(Observer)`和`subscribe(Subscriber)`,`subscribe()`
还支持不完整定义的回调,RxJava会自动根据定义创建出`Subscriber`.形式如下:
```Java
Action1<String> onNextAction = new Action1<String>() {
    // onNext()
    @Override
    public void call(String s) {
        Log.d(tag, s);
    }
};
Action1<Throwable> onErrorAction = new Action1<Throwable>() {
    // onError()
    @Override
    public void call(Throwable throwable) {
        // Error handling
    }
};
Action0 onCompletedAction = new Action0() {
    // onCompleted()
    @Override
    public void call() {
        Log.d(tag, "completed");
    }
};

// 自动创建 Subscriber ，并使用 onNextAction 来定义 onNext()
observable.subscribe(onNextAction);
// 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和 onError()
observable.subscribe(onNextAction, onErrorAction);
// 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
observable.subscribe(onNextAction, onErrorAction, onCompletedAction);
```
简单解释下这段代码中出现的`Action0`和`Action1`.`Action0`是RxJava的一个接口,它只有一个方法`call()`,这个方法是**无参无返回值的**;由于`onCompleted()`方法也是无参无返回值的,因此`Action0`可以被当成要给包装对象,将`onCompleted()`的内容打包起来将自己作为一个参数传入`subscribe()`以实现不完整定义的回调.这样其实也可以看作将`onCompleted()`方法作为参数传进了`subscribe()`,相当于其他语言中的闭包.`Action1`也是一个接口,它同样只有一个方法`call(T param)`,这个**也无返回值,但有一个参数**;与`Action0`同理,由于`onNext(T obj)`和`onError(Throwable error)`也是单参数无返回值的,因此`Action1`可以将`onNext(T obj)`和`onError(Throwable error)`打包起来传入`subscribe()`以实现不同完整定义的回调.事实上,虽然`Action0`和`Action1`在API中使用最广泛,但RxJava是提供了多个`ActionX`形式的接口(例如`Action2`,`Action3`)的,它们可以被用以包装不同的无返回值的方法.
> 注：正如前面所提到的，Observer 和 Subscriber 具有相同的角色，而且 Observer 在 subscribe() 过程中最终会被转换成 Subscriber 对象，因此，从这里开始，后面的描述我将用 Subscriber 来代替 Observer ，这样更加严谨。

**4.场景示例**
下面举两个例子:
a. **打印字符串数组**
将字符串数组names中的所有字符串依次打印出来:
```Java
String[] names = ...;
Observable.from(names)
    .subscribe(new Action1<String>() {
        @Override
        public void call(String name) {
            Log.d(tag, name);
        }
    });
```
b.由id取得图片并显示
由指定的一个drawable文件id`drawableRes`取得图片,并显示在`ImageView`中,并在出现异常的时候打印Toast报错.
```Java
int drawableRes = ...;
ImageView imageView = ...;
Observable.create(new OnSubscribe<Drawable>() {
    @Override
    public void call(Subscriber<? super Drawable> subscriber) {
        Drawable drawable = getTheme().getDrawable(drawableRes));
        subscriber.onNext(drawable);
        subscriber.onCompleted();
    }
}).subscribe(new Observer<Drawable>() {
    @Override
    public void onNext(Drawable drawable) {
        imageView.setImageDrawable(drawable);
    }

    @Override
    public void onCompleted() {
    }

    @Override
    public void onError(Throwable e) {
        Toast.makeText(activity, "Error!", Toast.LENGTH_SHORT).show();
    }
});
```
正如上面两个例子这样,创建出`Observable`和`Subscriber`,再用`subscribe()`将它们串起来,一次RxJava的基本使用就完成了,非常简单.
然而并没有什么卵用.
在RxJava的默认规则中,事件的发出和消费都是**在同一个线程的**.也就是说,如果只有上面的方法,实现出来的只是一个**同步观察者**.观察者模式本身的目的就是**后台处理,前台回调**的异步机制.因此异步对于RxJava是至关重要的.而要实现异步,则需要用到RxJava的另一个概念:`Scheduler`.
#### 线程控制--Scheduler(一)
在不指定线程的情况下,RxJava遵循的是**线程不变**的原则,即:在哪个线程调用`subscribe()`,就在哪个线程生产事件;在哪个线程生产事件,就这哪个线程消费事件.如果需要切换线程,就需要用到`Scheduler`.
**1.Scheduler的API(一)**
在RxJava中,`Scheduler`相当于线程控制器,RxJava通过它来指定每一段代码运行在什么样的线程.RxJava已经内置了几个`Scheduler`,它们已经适合大多数的使用场景:
* `Scheduler.immediate()`:直接在当前线程运行,相当于不指定线程.这是默认的`Scheduler`.
* `Scheduler.newThread()`:总是启用新线程,并在新线程执行操作.
* `Scheduler.io()`:I/O操作(读写文件,读写数据库,网络信息交换等)所使用的`Scheduler`.行为模式和`newThread()`差不多,区别在于`io()`的内部实现是一个无数量上限的线程池,可以重用空闲的线程,因此多数情况下`io()`比`newThread()`更有效率.不要把计算工作放在`io()`中,可以避免创建不必要的线程.
* `Scheduler.computation()`:计算所使用的`Scheduler`.这个计算指的是CPU密集型计算,即不会被I/O等操作限制性能的操作.例如图形的计算.
* 另外,Android还有一个专用的`AndroidScheduler.mainThread()`,它指定的操作将在Android主线程运行.
有了这几个`Scheduler`,就可以使用`subscribeOn()`和`observeOn()`两个方法来对线程进行控制了.
`subscribeOn()`指定`subscribe()`所发生的线程,即Observable.OnSubscribe被激活时所处的线程.或者叫做事件产生的线程
`observeOn()`:指定`Subscriber`所运行在的线程,或者叫做事件消费的线程.
文字不够直观,上代码:
```Java
Observable.just(1, 2, 3, 4)
    .subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
    .observeOn(AndroidSchedulers.mainThread()) // 指定 Subscriber 的回调发生在主线程
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer number) {
            Log.d(tag, "number:" + number);
        }
    });
```
上面这段代码中,由于`subscribeOn(Scheduler.io())`的指定,被创建的事件的内容`1`,`2`,`3`,`4`将会在IO线程发出;而由于`observeOn(AndroidSchedulers.mainThread())`的指定,因此`subscriber`数字的打印将发生在主线程.事实上,这种在`subscribe()`之前写上两句`subscribeOn(Sched.io())`和`observeOn(AndroidSchedulers.mainThread())`的使用方式非常常见.它适用于多数的**后台线程取数据,主线程显示**的程序策略.
而前面提到的由图片id取得图片并显示的例子,如果也加上这两句:
```Java
int drawableRes = ...;
ImageView imageView = ...;
Observable.create(new OnSubscribe<Drawable>() {
    @Override
    public void call(Subscriber<? super Drawable> subscriber) {
        Drawable drawable = getTheme().getDrawable(drawableRes));
        subscriber.onNext(drawable);
        subscriber.onCompleted();
    }
})
.subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
.observeOn(AndroidSchedulers.mainThread()) // 指定 Subscriber 的回调发生在主线程
.subscribe(new Observer<Drawable>() {
    @Override
    public void onNext(Drawable drawable) {
        imageView.setImageDrawable(drawable);
    }

    @Override
    public void onCompleted() {
    }

    @Override
    public void onError(Throwable e) {
        Toast.makeText(activity, "Error!", Toast.LENGTH_SHORT).show();
    }
});
```
那么,加载图片将会发生在IO线程,而设置图片则被设定在了主线程.这就意味着,即使加载图片耗费了几十甚至几百毫秒的时间,也不会造成丝毫界面的卡顿.
**2.Scheduler的原理(一)**
RxJava的Scheduler API很方便,也很神奇(加了一句话就把线程切换了,怎么做到的?而且`subscribe()`不是最外层直接调用的方法吗,它竟然也能被指定线程?).然而Scheduler的原理需要放在后面讲,因为它的原理是以下一节<变换>的原理作为基础的.
### 变换


