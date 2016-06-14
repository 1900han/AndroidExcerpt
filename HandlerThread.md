### 源码

```java
/**
 * Handy class for starting a new thread that has a looper. The looper can then be 
 * used to create handler classes. Note that start() must still be called.
 */
public class HandlerThread extends Thread {
  // 线程的优先级
    int mPriority;
  // 线程的id
    int mTid = -1;
  // 一个与Handler关联的Looper对象
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    /**
     * Constructs a HandlerThread.
     * @param name
     * @param priority The priority to run the thread at. The value supplied must be from 
     * {@link android.os.Process} and not from java.lang.Thread.
     */
    public HandlerThread(String name, int priority) {
        super(name);
      //设置优先级为默认线程
        mPriority = priority;
    }
    
    /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
      //获取当前线程的id
        mTid = Process.myTid();
      // 创建Looper对象
      // 这就是为什么我们要在调用线程的start()方法后才能得到Looper(Looper.myLooper不为null)
        Looper.prepare();
      // 同步代码快，当获得mLooper对象后，唤醒所有线程
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
      //设置线程优先级
        Process.setThreadPriority(mPriority);
      //Looper.loop之前在线程中需要处理的其他逻辑
        onLooperPrepared();
      //建立了消息循环
        Looper.loop();
      //一般执行不到这句，除非quit消息队列
        mTid = -1;
    }
    
    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason is isAlive() returns false, this method will return null. If this thread 
     * has been started, this method will block until the looper has been initialized.  
     * @return The looper.
     */
    public Looper getLooper() {
        if (!isAlive()) {
          //线程死了
            return null;
        }
        //同步代码块，正好和什么run方法中同步块对应
        //只要线程活着并且mLooper为null，则一直等待
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
          //退出消息循环
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            //退出消息循环
            looper.quitSafely();
            return true;
        }
        return false;
    }

    /**
     * Returns the identifier of this thread. See Process.myTid().
     */
    public int getThreadId() {
      //返回线程id
        return mTid;
    }
```

上面源码可以看到，HandlerThread主要是对Looper进行初始化，并提供一个Looper对象给新创建的Handler对象，使得Handler处理消息事件在子线程中处理，这样就发挥了Handler的优势，同时又可以很好的和线程结合到一起。

### HandlerThread实战

```java
public class ListenerActivity extends Activity {
    private HandlerThread mHandlerThread = null;
    private Handler mThreadHandler = null;
    private Handler mUiHandler = null;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.main);

        initData();
    }

    private void initData() {
        Log.i(null, "Main Thread id="+Thread.currentThread().getId());

        mHandlerThread = new HandlerThread("HandlerWorkThread");
        //必须在实例化mThreadHandler之前调运start方法，原因上面源码已经分析了
        mHandlerThread.start();
        //将当前mHandlerThread子线程的Looper传入mThreadHandler，使得
        //mThreadHandler的消息队列依赖于子线程（在子线程中执行）
        mThreadHandler = new Handler(mHandlerThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                Log.i(null, "在子线程中处理！id="+Thread.currentThread().getId());
                //从子线程往主线程发送消息
                mUiHandler.sendEmptyMessage(0);
            }
        };

        mUiHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                Log.i(null, "在UI主线程中处理！id="+Thread.currentThread().getId());
            }
        };
        //从主线程往子线程发送消息
        mThreadHandler.sendEmptyMessage(1);
    }
}
```

运行结果如下：

>Main Thread id=1
>
>在子线程中处理！id=113
>
>在UI主线程中处理！id=1

### 参考

[Android异步消息处理机制详解及源码分析](http://blog.csdn.net/yanbober/article/details/45936145)

[Android HandlerThread 完全解析](http://blog.csdn.net/lmj623565791/article/details/47079737)