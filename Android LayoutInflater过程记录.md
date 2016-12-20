### 常见用法
```java
LayoutInflater layoutInflater = LayoutInflater.from(context);
```
还有一种
```java
LayoutInflater layoutInflater = (LayoutInflater)context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```
得到了LayoutInflater的实例之后,可以调用它的`inflate()`方法来加载布局
```java
layoutInflater.inflate(resourceId,root);
```
inflate()方法一般接收两个参数,
**第一个参数是要加载的布局id,**
**第二个参数是指给该布局的外部再嵌套一层父布局,如果不需要就直接传null.**
这样就成功创建了一个布局的实例,之后再将它添加到指定的位置就可以显示出来了.
下面进入源码的世界里看看,到底是个什么样的过程?
不管是使用哪个`inflate()`方法的重载,最终都会调用到`LayoutInflater`的如下代码中:
```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();

                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (Exception e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                                + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }
            return result;
        }
    }
```
首先声明了View result = root; //最终返回值为result
下面执行了temp = createViewFromTag(root,name,attrs);创建了View
```java
if(root!=null){
    params = root.generateLayoutParams(atts);
    if(!attachToRoot){
        temp.setLayoutParams(params);
    }
}
```
可以看到,当root不为null,attachToRoot为false时,为temp设置了LayoutParams.
继续往下,看
```java
if(root!=null&&attachToRoot){
    root.addView(temp,params);
}
```
当root不为null,attachToRoot为true时,将tmp按照params添加到root中.
然后
```java
if(root==null||!attachToRoot){
    result = temp;
}
```
如果root为null,或者attachToRoot为false则,将temp赋值给result.
最后返回result.
从上面的分析可以看出:
Inflate(resId,null)只创建temp,返回temp;
Inflate(resId,parent,false)创建temp,然后执行temp.setLayoutParams(params);返回temp
Inflate(resId,parent,true)创建temp,然后执行root.addView(temp,params);最后返回root
由上面已经能够解释
Inflate(resId,null)不能正确处理宽和高是因为:layout_width,layout_height是相对了父级设置的,必须与父级的LayoutParams一致.而此temp的getLayoutParams为null.
Inflate(resId,parent,false)可以正确处理,因为temp.setLayoutParams(params),这个params正是root.generateLayoutParams(attrs);得到的.
inflate(resId,parent,true)不仅能正确的处理,而且已经把resId这个View加入到了parent,并且返回的是parent,和以上两者返回值有绝对区别.
