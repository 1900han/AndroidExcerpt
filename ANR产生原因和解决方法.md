在Android世界里,APP的响应能力是由**ActivityManagerService**和**WindowManagerService**来监控的,通常在如下三种情况下会弹出ANR对话框.
* 5s内无法响应用户输入事件(键盘输入,触摸屏幕)
* BroadcastReceiver在10s内无法结束
* 主线程在执行Service的各个生命周期函数时20s内没有执行完毕
造成上面的情况**首要原因就是在主线程里面做了太多的阻塞耗时操作**
### 如何避免ANR
> 不要在主线程里面做繁重的操作
### ANR的处理
1. 主线程阻塞的
        开辟单独的子线程来处理耗时阻塞事务
2. CPU满负荷,I/O阻塞的
        I/O阻塞一般来说就是文件读写或数据库操作执行在主线程了, 也可以通过开辟子线程的方式异步执行.
3. 内存不够用的
        增大VM内存,使用largeHeap属性,排查内存泄漏
        

        