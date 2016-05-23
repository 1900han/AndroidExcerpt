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

* public synchronized void execute(final Runnable r)中的参数r正是从exec.execute(mFuture)传递过来的mFuture，后面将介绍mFuture
* mTasks创建了一个队列，根据`offer`函数的注释可以知道这个队列用来保存要执行的任务，即每次调用AsyncTask要执行的任务，整个任务将当作一个参数传入到offer中
* `execute`这个函数的整个逻辑都是在一个线程里面完成的
* `scheduleNext()`方法中系统先从mTasks整个任务队列里面拿出一个任务，然后使用THREAD_POOL_EXECUTOR线程池执行这个任务。

综合上面的几点，梳理一下流程，首先，要执行的任务被放进了mTasks，然后子啊scheduleNext中会去取出一个任务去执行，那么什么时候会调用scheduleNext，只有在任务队列取出元素为null（即任务队列中没有任务）和上一个任务执行完毕之后执行finally块中的代码才会调用scheduleNext，这**证实了3.0之后的AsyncTask默认是单个后台任务串行的方式执行任务的**。

接下来看THREAD_POOL_EXECUTOR.execute(mActive)







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





### 参考

[ Android AsyncTask 源码解析](http://blog.csdn.net/lmj623565791/article/details/38614699)

[Android AsyncTask完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/11711405)

[AsyncTask使用及源码解析](http://githubonepiece.github.io/2015/12/12/AsyncTask/)

[关于Android中工作者线程的思考](http://droidyue.com/blog/2015/12/20/worker-thread-in-android/index.html)

[Android之AsyncTask介绍](http://wangkuiwu.github.io/2014/06/25/AsyncTask/)