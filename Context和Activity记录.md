![Context类图](http://o75vlu0to.bkt.clouddn.com/Context%20class%20diagram.png)

### Context

* 描述的是一个应用程序环境的信息，即上下文。
* 该类是一个抽象(abstract class)类，Android提供了该抽象类的具体实现类(后面我们会讲到是ContextIml类)
* 通过它我们可以获取应用程序的资源和类，也包括一些应用级别操作，例如：启动一个Activity，发送广播，接受Intent信息 等

**一个应用程序有几个Context**

Context数量=Activity数量+Service数量+1(**Application对应的Context实例**)

**Context作用域**

![Context作用域](http://upload-images.jianshu.io/upload_images/1187237-fb32b0f992da4781.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总结：凡是跟UI相关的，都应该使用Activity做为Context来处理；其他的一些操作，Service,Activity,Application等实例都可以

**如何获取Context**

1. View.getContext 返回当前View对象的Context对象，通常是当前正在展示的Activity对象
2. Activity.getApplicationContext ,获取当前Activity所在的(应用)进程的Context对象
3. Activity.this返回当前的Activit实例，如果是UI控件需要使用Activity作为Context对象，但是默认的Toast实际上使用ApplicationContext也可以
4. ContextWrqpper.getBaseContext() 取一个ContextWrapper进行装饰之前的Context，可以使用这个方法，这个方法在实际开发中使用并不多，也不建议使用。

**getApplication()和getApplicationContext()**

两者的内存地址都是相同的,它们是同一个对象。``getApplication()``只能在Activity和Service才能调用，而BroadcastReceiver中只能借助``getApplicationContext()``方法。

**使用Context的正确姿势**

* Application的Context能搞定的，优先使用Application的Context
* 不要让生命周期长于Activity的对象持有到Activity的引用
* 尽量不要在Activity中使用非静态内部类，因为非静态内部类会隐式的持有外部实例的引用，如果使用静态内部类，将外部实例引用作为**弱引用**持有。



### Activity为什么划分这么多生命周期？

1. 提供流畅的体验
2. 异常状态保存数据

* 创建与销毁 ``onCreate()``，``onDestroy()``
* 是否可见``onStart()``，``onStop()`` onStart表示应用已经可见，但只运行在后台，没到前台。onStop表示Activity运行在后天。当用户使用透明主题，不会调用onStop。
* 是否在前台``onResume()``，``onPause()`` 这两个是从activity是否位于前台来回调的，是一组

**Activity的``onSaveInstanceState()``与``onRestoreInstanceState()``的作用**

在资源紧张的情况下，系统会选择杀死一些处于非栈顶的Activity来回收资源。为了能够让这些可能被杀死的Activity能够在恢复显示的时候状态不丢失，所以需要在Activity从栈顶往下压的时候提供``onSaveInstanceState()``的回调用来提前保存状态信息。

**``onSaveInstanceState()``与``onRestoreInstanceState()``的调用时机**

只要某个Activity是做**入栈并且非栈顶**时（启动跳转其他Activity或者点击Home按钮），此Activity是需要调用``onSaveInstanceState()``的，如果Activity是做**出栈**的动作（点击**back**或者**执行finish**），是**不会调用**``onSaveInstanceState``的。

只有在Activity真的**被系统非正常杀死过**，恢复显示Activity的时候，就会调用onRestoreInstanceState().

**``onPause()``和``onStop()``区别**

pause是**失去焦点**，stop是**不可见**

#### 如何判断APP在前台？

根据生命周期，因为Android中只有一个Activity在运行

### 参考

[Context都没弄明白，还怎么做Android开发？](http://www.jianshu.com/p/94e0f9ab3f1d#)

[Android源码分析-全面理解Context](http://blog.csdn.net/singwhatiwanna/article/details/21829971)

[胡凯](http://hukai.me/android-activitylifecycle-onsaveinstancestate/)