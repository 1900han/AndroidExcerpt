### 简单使用AsyncTask

```java
public abstract class AsyncTask<Params,Progress,Result>{
  
}
```

AsyncTask是一个**泛型的抽象类**，意味着使用时需要写一个类去**继承**它，在继承时要为AsyncTask类指定三个泛型参数，这三个参数的用途如下：

1. Params

   在执行AsyncTask时需要传入的参数，可用于在后台任务中使用

2. Progress

   后台任务执行时，如果需要在界面上显示当前的进度，则使用这里指定的泛型作为进度单位

3. Result

   当任务执行完毕后，如果需要对结果进行返回，则使用这里指定的泛型作为返回值类型

接下来一般要重写AsyncTask中的几个方法才能进行使用

1. `onPreExecute()`

   这个方法会在后台任务开始执行之间调用，用于进行一些界面上的初始化操作，比如显示一个进度对话框等

2. `doInBackground(Params...)`

   这个方法中的使用代码都会在**子线程**中运行，应该在这里**处理所有的耗时任务**。任务一旦完成可以通过return语句来讲任务的执行结果进行返回，如果AsyncTask的第三个泛型参数指定的是Void，就可以不返回任务执行结果。注意，在这个方法中是**不可以进行UI操作的**，如果需要更新UI元素，比如反馈当前任务的进度，可以调用`publishProgress(Progress...)`方法来完成。

3. `onProgerssUpdate(Progress...)`

   当在后台任务中调用了`publishProgress(Profress...)`方法后，这个方法就很快被调用，方法中携带的参数就是在后台任务中传递过来的。在这个方法中可以对UI进行操作，利用参数中的数值就可以对界面元素进行相应的更新。

4. `onPostExecute(Result)`

   当后台任务执行完毕并通过return语句进行返回时，这个方法就很快被调用。返回的数据会作为参数传递到此方法中，可以利用返回的数据来进行一些UI操作，比如说提醒任务执行的结果，以及关闭掉进度条对话框等。

完整的Demo示例：

```java
class DownloadTask extends AsyncTask<Void,Integer,Boolean>{
  @Override
  protected void onPreExecute(){
    progressDialog.show();
  }
  
  @Override
  protected Boolean doInBackground(Void... params){
    try{
      while(true){
        int downloadPercent = doDownload();
        publishProgress(downloadPerenct);
        if(downloadPerenct >= 100){
          break;
  		}
      }
 	} catch(Exception e){
      return false;
	}
    return true;
 }
  @Override
  protected void onProgressUpdate(Integer... values){
    progressDialog.setMessage("当前下载进度:"+values[0]+"%");
  }
  @Override
  protected void onPostExecute(Boolean result){
    progressDialog.dismiss();
    if(result){
      Toast.makeText(context,"下载成功",Toast.LENGTH_SHORT).show();
    } else {
      Toast.makeText(context,"下载失败",Toast.LENGTH_SHORT).show();
	}
  }
}
```

想启动这个任务，只需要简单的调用一下代码：

`new DownloadTask().execute();`

### 分析AsyncTask源码

在3.0后，AsyncTask有很大变动。

从异步任务的起点开始，进入`execute`方法：

```java
public final AsyncTask<Params,Progress,Result> execute(Params... params){
  return executeOnExexutor(sDefaultExecutor , params);
}

public final AsyncTask<Params,Progress,Result> executeOnExecutor(Executor exec,Params... params){
  if(mStatus != Status.PENDING){
  	 switch (mStatus) {
  	     case RUNNING:
 			 throw new IllegalStateException("Cannot execute task: the task is already 						running.");
 		 case FINISHED:
 		 	 throw new IllegalStateException("Cannot execute task:the task has already 						been executed (a task can be executed only once)");
  	 }
  }
  
  mStatus = Status.RUNNING;
  
  onPreExecute();
  
  mWorker.mParams = params;
  exec.execute(mFuture);
  return this;
}
```

* 设置当前AsyncTask的状态为RUNNING，上面的switch可以看出，**每个异步任务在完成前只能执行一次**


* 执行了`onPreExecute`，当前依然在**UI线程**，所以我们可以在其中做一些准备工作;也证明`onPreExecute`**第一个执行**


* 将传入的参数赋值给了mWorker.mParams


* exec.execute(mFuture)

这里exec为`executeOnExecutor(sDefaultExecutor,params)`中的`sDefaultExecutor`

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor{
  final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
  Runnable mActive;
  
  public synchronized void execute(final Runnable r){
      mTasks.offer(new Runnable() {
  		  public void run(){
  			  try {
  				  r.run();
			  } finally {
  				  scheduleNext();
			  }
		  }
	  });
      if (mActive == null) {
  		  scheduleNext();
	  }
  }
  
  protected synchronized void scheduleNext() {
  	  if ((mActive = mTasks.poll()) != null) {
  		  THREAD_POOL_EXECUTOR.execute(mActive);
	  }
  }
}
```

上面代码可以看出下面几点：

* public synchronized void execute(final Runnable r)中的参数r正是从exec.execute(mFuture)传递过来的**mFuture**，后面将介绍mFuture
* mTasks创建了一个队列，根据`offer`函数的注释可以知道这个队列用来保存要执行的任务，即每次调用AsyncTask要执行的任务，整个任务将当作一个参数传入到offer中
* `execute`这个函数的整个逻辑都是在一个线程里面完成的
* `scheduleNext()`方法中系统先从mTasks整个任务队列里面拿出一个任务，然后使用THREAD_POOL_EXECUTOR线程池执行这个任务。

综合上面的几点，梳理一下流程，首先，要执行的任务被放进了mTasks，然后子啊scheduleNext中会去取出一个任务去执行，那么什么时候会调用scheduleNext，只有在任务队列取出元素为null（即任务队列中没有任务）和上一个任务执行完毕之后执行finally块中的代码才会调用scheduleNext，这**证实了3.0之后的AsyncTask默认是单个后台任务串行的方式执行任务的**。

关注下**THREAD_POOL_EXECUTOR**

```java
public static final Executor THREAD_POOL_EXECUTOR = 
       new ThreadPoolExecute(CORE_POOL_SIZE,MAXIMUM_POOL_SIZE,KEEP_ALIVE,
  		TimeUnit,SECONDS,sPoolWorkQueue,sThreadFactory);
```

```java
private static final int CORE_POOL_SIZE = 5;
private static final int MAXIMUM_POOL_SIZE = 128;
private static final int KEEP_ALIVE = 1;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
   private final AtomicInteger mCount = new AtomicInteger(1);

      public Thread newThread(Runnable r) {
         return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
      }
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(10);
```

可以看到如果此时这里有10个任务同时调用execute(),第一任务入队，然后在mActive = mTask.poll()!=null被取出，并且赋值给mActive，然后交给线程池去执行。然后第二个任务入队，但是此时mActive并不为null，并不会执行scheduleNext();所以如果第一个任务比较慢，10个任务都会进入队列等待；真正执行下一个任务的时机是，线程池完成第一个任务后，调用Runnable中的finally代码块中的scheduleNext，所以虽然内部有要给线程池，**其实调用的过程还是线性的，一个接一个的执行，相当于单线程**。

上面的任务在运行时就会执行`r.run()`，而实际上调用的FutureTask中`run()`方法，这个run方法会执行Callable的`call()`方法，即会执行到mWorker的call方法。

这里需要对mWorker和mFuture进行了解

mWorker类的定义

```java
private static abstract class WorkerRunnable<Params,Result> implements Callable<Result>{
  Params[] mParams;
}
```

发现这是Callable的子类，且包含一个mParams用于保存传入的参数，看初始化mWorker的代码：

```java
public AsyncTask(){
  mWorker = new WorkerRunnable<Params ,Result>(){
      public Result call() throws Exception {
  		 mTaskInvoked.set(true);
         Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
         //noinspection unchecked
         return postResult(doInBackground(mParams));
	  }
  };
}
```

可以看到mWorker在AsyncTask的构造函数中完成了初始化，并且因为是一个抽象类，在这里new了一个实现类，实现了`call()`方法，call方法设置了**mTaskInvoked = true**，且最终调用`doInBackground(mParams)`方法，并返回**Result**值作为参数给`postResult()`

继续看`postResult()`

```java
private Result postResult(Result result){
  @SuppressWarnings("unchecked")
  Message message = sHandler.obtaninMessage(MESSAGE_POST_RESULT,new 	AsyncTaskResult<Result>(this,result));
}
```

这里使用了异步消息机制，传递了一个消息message，message.what为MESSAGE_POST_RESULT;

message.object = new AsyncTaskResult(this,result);

```java
private static class AsyncTaskResult<Data> {
  final AsyncTask mTask;
  final Data[] mData;
  
  AsyncTaskResult(AsyncTask task,Data... data){
    mTask = task;
    mData = data;
  }
}
```

AsyncTaskResult就是一个简单的携带参数的对象。

此时就可以肯定在某处存在一个**sHandler**，且复写了其handleMessage方法等待消息的传入，以及消息的处理。

```java
private static final IntennalHandler extends Handler{
  @SuppressWarning({"unchecked","RawUseOfParameterizedType"})
  @Override
  public void handleMessage(Message msg){
    AsyncTaskResult result = (AsyncTaskResult) msg.obj;
    switch (msg.what) {
      case MESSAGE_POST_RESULT:
        // There is only the result
        result.mTask.finish(result.mData[0]);
        break;
      case MESSAGR_POST_PROGRESS:
        result.mTask.onProgressUpdate(result.mData);
        break;
    }
  }
}
```

接收到MESSAGE_POST_RESULT消息时，执行了result.mTask.finish(result.mData[0]);即等于AsyncTask.this.finish(result)

```java
private void finish(Result result){
  if(isCancelled()){
     onCancelled(result);
  }else {
    onPostExecute(result);
  }
  mStatus = Status.FINISHED;
}
```

如果调用了`cancel()`则执行onCancelled回调；正常执行的情况下调用`onPostExecute(result)`;**这里的调用时在handler的handleMessage中，所以是在UI线程中。**

最终讲状态置为**FINISHED**。

再接着看mFuture，依然时在AsyncTask的构造方法中完成初始化，将**mWorker**作为参数，复写了其**done()**方法。

```java
public AsyncTask(){
  ...
  mFuture = new FutureTask<Result>(mWorker){
     @Override 
     protected void done(){
        try {
            postResultIfNotInvoked(get());
        } catch (InterruptedException e){
  			android.util.Log.w(LOG_TAG,e);
        } catch (ExecutionException e){
            throw new RuntimeException("An error occured while executing doInBackgroun",e.getCause());
		} catch (CancellationException e){
  			postResultIfNotInvoked(null);
		}
     }
  };
}
```

任务执行结束会调用：postResultIfNotInvoked(get());get()表示获取mWorker的call的返回值，即Result

```java
private void postResultIfNotInvoked(Result result) {
  final boolean wasTaskInvoked = mTaskInvoked.get();
  if (!wasTaskInvoked) {
     postResult(result);
  }
}
```

如果mTaskInvoked不为true，则执行postResult；但是在**mWorker初始化时就已经将mTaskInvoked为true**，所以一般这个postResult执行不到。

最后一个方法`publishProgress()`

```java
protected final void publishProgress(Progress... values){
  if(!isCancelled()){
     sHandler.obtainMessage(MESSAGE_POST_PROGESS,
                            new AsyncTaskResult<Progress>(this,values)).sendToTarget();
  }
}
```

这个方法很简单，直接使用sHandler发送一个消息，携带传入的值。

```java
private static class InternalHandler extends Handler {
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult result = (AsyncTaskResult) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                 // There is only one result
                 result.mTask.finish(result.mData[0]);
                 break;
            case MESSAGE_POST_PROGRESS:
                 result.mTask.onProgressUpdate(result.mData);
                 break;
        }
    }
}
```

在handleMessage中进行了我们的onProgressUpdate(Result.mData)的调用。

### 缺陷

可以分为两个部分说，在3.0以前，最大支持128个线程的并发，10个任务的等待。在3.0以后，无论有多少任务，都会在其内部**单线程**执行。

### 参考

[ Android AsyncTask 源码解析](http://blog.csdn.net/lmj623565791/article/details/38614699)

[Android AsyncTask完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/11711405)

[AsyncTask使用及源码解析](http://githubonepiece.github.io/2015/12/12/AsyncTask/)

[关于Android中工作者线程的思考](http://droidyue.com/blog/2015/12/20/worker-thread-in-android/index.html)

[Android之AsyncTask介绍](http://wangkuiwu.github.io/2014/06/25/AsyncTask/)

[AsyncTask的介绍和使用](http://www.muzileecoding.com/androidsource/Android-AsyncTask.html)

