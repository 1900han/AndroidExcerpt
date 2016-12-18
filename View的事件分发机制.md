### 重点关注的对象和方法

主要关注对象是**_MotionEvent_**，即点击事件。点击事件的分发过程主要由三个很重要的方法共同完成。

**`public boolean dispatchTouchEvent(MotionEvent event)`**

用来进行事件的分发。如果事件能够传递给当前的View，那么此方法**一定会被调用**，返回结果受当前View的**`onTouchEvent`**和**下级View**的**`dispatchTouchEvent`**方法的影响，表示是否消耗当前事件。**_在TextView等这种最小的View中不会有该方法。_**

**`public boolean onInterceptTouchEvent(MotionEvent event)`**

**在上述方法内部调用**，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列当中，**此方法不会被再次调用**，返回结果表示是否拦截当前事件。

**`public boolean onTouchEvent(MotionEvent event)`**

**在`dispatchTouchEvent`方法中调用**，用来**_处理点击事件_**，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接受到事件。

之间的关系可以用下面的伪代码表示：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
	boolean consume = false;
	if(onInterceptTouchEvent(event)){
		consume = onTouchEvent(event);
    } else {
		consume = child.dispatchTouchEvent(event);
    }
	return consume;
}
```

### View中触摸事件的概述

View中的`dispatchTouchEvent()`会将事件传递给“自己的`onTouch()`”,"自己的`onTouchEvent()`"进行处理。而且**`onTouch()`**的**优先级**比**`onTouchEvent()`**的优先级要**高**。

`onTouch()`与`onTouchEvent()`都是View中用户处理触摸事件的API。`onTouch()`是OnTouchListener接口中的函数，onTouchListener接口需要用户自己实现。`onTouchEvent()`是View自带的接口，Android系统提供了默认的实现；当然，用户可以重载该API。

**`onTouch()`**与**`onTouchEvent()`**有两个不同之处：

* `onTouch()`是View提供给用户，让用户自己处理触摸事件的接口。而`onTouchEvent()`是Android系统自己实现的接口。
* `onTouch()`的优先级比`onTouchEvent()`的高。

`dispatchTouchEvent()`中分发事件的时候，会先将事件分配给`onTouch()`进行处理，然后才分配给	`onTouchEvent()`进行处理。如果`onTouch()`对触摸事件进行了处理，并且返回**true**；那么，该触摸事件就**不会分配**给`onTouchEvent()`进行处理了。只有当`onTouch()`**没有处理**，或者**处理了返回false时**，才会分配给`onTouchEvent()`进行处理。
**View中的事件调度顺序**
onTouchListener-->onTouchEvent-->onLongClickListener-->onClickListener
