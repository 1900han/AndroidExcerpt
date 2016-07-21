## res/raw和assets的区别

**相同点**

两者目录下的文件在打包后会原封不动的保存在apk包中,不会被编译成二进制.

**不同点**

1. res/raw中的文件会被映射到R.java文件中,访问的时候直接使用资源ID即R.id.filename;assets文件夹下的文件不会被映射到R.java中,访问的时候需要AssetManager类
2. **res/raw*不可以有目录结构***,而**assets**则**可以有目录结构**,也就是assets目录下可以再建立文件夹

##　读取文件资源

1. 读取res/raw下的文件资源,通过以下方式获取输入流来进行写操作

   ```java
   InputStream is = getResources().openRawResource(R.id.filename);
   ```

2. 读取assets下的文件资源,通过以下方式获取输入流来进行写操作

   ```java
   AssetsManager am = null;
   am = getAssets();
   InputStream is = am.open("filename");
   ```

## 参考

[Android中资源文件夹res/raw和assets的使用](http://blog.csdn.net/zuolongsnail/article/details/6444806)

