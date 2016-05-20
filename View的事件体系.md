![View的位置坐标和父容器的关系](http://o75vlu0to.bkt.clouddn.com/View%E7%9A%84%E4%BD%8D%E7%BD%AE%E5%9D%90%E6%A0%87%E5%92%8C%E7%88%B6%E5%AE%B9%E5%99%A8%E7%9A%84%E5%85%B3%E7%B3%BB.jpg)

#### **View的宽高和坐标的关系**

width = right - left

height = bottom - top

#### **x,y,translationX,translationY**

x,y是View左上角的坐标

translationX , translationY是View左上角相对于父容器的偏移量。

x = left + translationX

y = top + translationY

View在平移的过程中，top和left表示的是原始左上角的位置信息，**其值并不会发生改变**，此时发生改变的事x,y,translationX和translationY这四个参数。

#### **MotionEvent 和 TouchSlop**

MotionEvent 事件类型

* ACTION_DOWN
* ACTION_MOVE
* ACTION_UP

TouchSlop是系统所能识别出的被认为是滑动的最小距离。

**ViewConfiguration.get(getContext90).getScaledTouchSlop()**

#### **View的滑动**

* 使用**scrollTo** / **scrollBy**

  `scrollBy()`实际上是调用了`scrollTo()`,它实现了基于当前位置的**相对滑动**，而scrollTo则实现了基于所传递参数的**绝对滑动**。滑动过程中改变View内部的两个属性**_mScrollX_**和**_mScrollY_**，这两个属性可以通过_**getScrollX**_和**_getScrollY_**分别获得。**mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离**，而**mScrollY的值总是等于View上边缘和View内容上边缘在竖直方向的距离**。

  View边缘是指View的位置，由四个顶点组成，而View内容边缘是指View中的内容的边缘，scrollTo和scrollBy**只能改变View内容的位置而不能改变View在布局中的位置**。

  mScrollX和mScrollY的单位为**像素**，并且当View左边缘在View内容左边缘的右边时，mScrollX为正值，反之为负值；当View上边缘在View内容上边缘的下边时，mScrollY为正值，反之为负值。

* 使用动画

  View动画是对View的影像做操作，并不能真正改变View的位置参数，包括宽/高，并且如果希望动画后的效状态能够保留还必须将**fillAfter**属性设置为**true**，否则动画完成后效果会消失。使用**属性动画**不会存在上述问题，但是在 **Android 3.0**以下无法使用属性动画。

* 改变布局参数，即改变**LayoutParams**

**总结:**

* scrollTo/scrollBy: 操作简单，适合对View内容的滑动;
* 动画:操作简单，主要使用于没有交互的View和实现复杂的动画效果;
* 改变布局参数: 操作稍微复杂，适用于有交互的View。

#### **弹性滑动**

* 使用Scroller

  Scroller本身**并不能实现**View的滑动，它需要配合View的**computeScroll**方法才能完成弹性滑动的效果，它不断地让View重绘，而每一次重绘距滑动起始时间会有一个时间间隔，通过这个时间间隔Scroller就可以得出View当前的滑动位置，知道了滑动位置就可以通过scrollTo方法完成View的滑动。

* 使用动画

* 使用延时策略

