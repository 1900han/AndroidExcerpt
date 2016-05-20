#### 常见的滑动冲突场景

* 外部滑动方向和内部滑动方向**不一致**
* 外部滑动方向和内部滑动方向**一致**
* 上面两种情况的嵌套

#### 滑动冲突的处理规则

对于场景1，处理规则是: 当用户左右滑动时,需要让外部的View拦截点击事件,当用户上下滑动时,需要让内部View拦截点击事件.这个时候可以根据特征来解决滑动冲突,具体来说就是:根据滑动时水平滑动还是竖直滑动来判断到底由谁来拦截事件.根据下图所示,根据滑动过程中两个点之间的坐标就可以得出到底是水平滑动还是竖直滑动.如何根据坐标来得到滑动的方向呢?**可以依据滑动路径和水平方向所形成的夹角;也可以根据水平方向和竖直方向上的距离差来判断,某些特殊时候还可以根据水平和竖直方向的速度差来做判断.**

![滑动过程示意](http://o75vlu0to.bkt.clouddn.com/%E6%BB%91%E5%8A%A8%E8%BF%87%E7%A8%8B%E7%A4%BA%E6%84%8F.png)

场景2来说，比较特殊，无法根据场景1中的方法判断，但这个时候一般可以在**业务**上找到突破点。

场景3，滑动规则更加复杂，和场景2一样，只能再业务上寻找突破点。

#### 滑动冲突的解决方式

1. ##### 外部拦截法

   指点击事情都先经过父容器的拦截处理，如果父容器需要此事件就拦截，如果不需要此事件就不拦截，这样就可以解决滑动冲突的问题，这种方法符合点击事件的分发机制。外部拦截法需要重写父容器的**`onInterceptTouchEvent()`**方法，在内部做相应的拦截即可。

   ```java
   public boolean onInterceptTouchEvent(MotionEvent event){
     boolean intercepted = false;
     int x = (int) event.getX();
     int y = (int) event.getY();
     switch (event.getAction()){
       case MotionEvent.ACTION_DOWN:{
         // 父容器必须返回false，既不拦截ACTION_DOWN事件，因为父容器一旦拦截了ACTION_DOWN，那么后续的ACTION_MOVE和ACTION_UP事件都会交由父容器处理，这个时候事件没法再传给子元素
     		intercepted = fasle;
        	break;
   	}
       case MotionEvent.ACTION_MOVE:{
         // 根据需求决定是否拦截，需要返回true，否则返回false
     		if(父容器需要当前点击事件)｛
       		intercepted = true;
   		} else{
    			intercepted = false;
   		}
        	break;
   	}
     case MotionEvent.ACTION_UP:{
       // 此处必须返回false，因为ACTION_UP事件本身没有太多意义
     		intercepted = false;
       	break;
   	}
     default:
     	break;
     }
     mLastXIntercept = x;
     mLastYIntercept = y;
     return intercepted;
   }
   ```

   ​