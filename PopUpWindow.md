###构造函数
![](http://o75vlu0to.bkt.clouddn.com/PopupWindow%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0.png)
###显示PopupWindow
```table
方法|简洁
showAsDropDown(View anchor)|相对某个控件的位置（**正左下方**），无偏移
showAsDropDown(View anchor, int xoff, int yoff, int gravity)|
showAsDropDown(View anchor, int xoff, int yoff)|相对某个控件的位置，有偏移（正数表示下方右边，负数表示（上方左边））
showAtLocation(View parent, int gravity, int x, int y)|父容器容器相对位置，例如正中央Gravity.CENTER，下方Gravity.BOTTOM等
```
###设置大小
* 调用有宽高参数的构造函数
```java
LayoutInflater inflater = (LayoutInflater)getSystemService(Context.LAYOUT_INFLATER_SERVICE);
View contentview = inflater.inflate(R.layout.popup_process, null);
PopupWindow popupWindow = new PopupWindow(contentview,LayoutParams.WRAP_CONTENT,
LayoutParams.WRAP_CONTENT);
```
* 通过setWidth和setHeight设置 
```java
PopupWindow popupWindow = new PopupWindow(contentview);
popupWindow.setWidth(LayoutParams.WRAP_CONTENT);           
popupWindow.setHeight(LayoutParams.WRAP_CONTENT);
```
两种方法是等效的,不管采用何种方法,必须设置宽和高,否则不显示任何东西.
上面的wrap_content可以换成match_parent也可以是具体数值,它是指PopupWindow的大小,也就是从contentView的大小,注意PopupWindow根据这个大小显示你的View,**如果你的View本身是从XML得到的,那么XML的第一层view的大小属性将被忽略.相当于PopupWindow的width和height属性直接和第一层view相对应.**
###PopupWindow的焦点
setFocusable(true),则PopupWindow本身可以看作一个类似于对话框的东西(但有区别),PopupWindow弹出后,所有的触屏和物理按键都有PopupWindow处理.其他任何事件的响应都必须发生在PopupWindow消失之后,(home等系统层面的事件除外).
例如,一个PopupWindow出现的时候,按back键首先是让PopupWindow消失,第二次按才是退出Activity,准确的说是想退出Activity得首先让PopupWindow消失,因为并不是任何情况下按back PopupWindow都会消失,**必须在PopupWindow设置了背景的情况下.**
**如果PopupWindow中有Editor的话,focusable要为true.**
###背景是否为空对Touch事件的影响
如果有背景,则会在contentView外面包一层PopupViewContainer之后作为mPopupView,如果没有背景,则直接用contentView作为mPopupView.
而这个PopupViewContainer是一个内部私有类,它继承了FrameLayout,在其中重写了Key和Touch事件的分发处理
![](http://o75vlu0to.bkt.clouddn.com/PopupViewContainer-dispatchEvent.png)
**由于PopupView本身并没有重写Key和Touch事件的处理，所以如果没有包这个外层容器类，点击Back键或者外部区域是不会导致弹框消失的.**