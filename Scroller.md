任何一个控件都是可以_**滚动**_的,因为在View类中有`scrollTo()`和`scrollBy()`方法.但是滑动的是view的**内容**而不是view本身.

![scrollTo](http://o75vlu0to.bkt.clouddn.com/scrollTo.png)

![scrollBy](http://o75vlu0to.bkt.clouddn.com/scrollBy.png)

### 两者区别

*scrollBy()*是让View相当于**当前**的位置滚动某段距离

scrollTo()是让View相对于**初始**的位置滚动某段距离

**需要注意，就是两个scroll方法中传入的参数，第一个参数x表示相对于当前位置横向移动的距离，正值向左移动，负值向右移动，单位是像素。第二个参数y表示相对于当前位置纵向移动的距离，正值向上移动，负值向下移动，单位是像素.**

