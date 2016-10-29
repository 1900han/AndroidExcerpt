### URL配置

> @GET("users/{user}/repos")
>
> Call<List<Repo>> listRepos(@Path("user") String user);

* **path**是**绝对路径**的形式,**baseUrl**是**文件形式**

  path="/apath",baseUrl = "http://host:port/a/b"

  Url = "http://host:port/apath"

* **path**是**相对路径**,**baseUrl**是**目录形式**:

  path="apath" ,baseUrl="http://host:port/a/b/"

  Url = "http://host:port/a/b/apath"

**优先选择第二种**

### 参数类型

* Query&QueryMap
* Field&FieldMap
* Part&PartMap

### Convert

* RequestBodyConverter
* ResponseBodyConverter


### 使用步骤

```java
public interface GitHubSevice{
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

```java
Retrofit retrofit = new Retrofit.Builder().baseUrl("https://api.github.com/").build();
GithubService service = retrofit.create(GitHubService.class);
```

```java
Call<List<Repo>> repos = service.listRepos("octocat");
List<Repo> = repos.excute();
```

根据上面的代码进入对应的源码:

**Retrofit.Builder**是一个**静态内部类**,下面是这个类的**属性**

```java
private Platform platform;
private okhttp3.Call.Factory callFactory;
private HttpUrl baseUrl;
private List<Converter.Factory> converterFactories = new ArrayList<>();
private List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
private Executor callbackExecutor;
private boolean validateEagerly;
```

主要是用来**配置Retrofit**,因为**Retrofit**也具有**同样的属性**.

retrofit.create(GitHubService.class),进入到Retrofit类中查看`create()`

![retofit.create()](http://o75vlu0to.bkt.clouddn.com/retrofit.create.png)

这里使用了**动态代理**,当使用GithubService中的方法,会变成由代理类调用`invoke()`.这个方法中的代码很重要.

Retofit中`loadServiceMethod()`

![](http://o75vlu0to.bkt.clouddn.com/Retrofit.loadServiceMethod.png)

这里使用了**缓存**,毕竟每次重新调用耗费资源.这个引出了**ServiceMethod**类,内部也由一个**静态内部类Builder**,下面是Builder的构造方法

```java
public Builder(Retrofit retrofit, Method method) {
       this.retrofit = retrofit;
       this.method = method;
       this.methodAnnotations = method.getAnnotations();
       this.parameterTypes = method.getGenericParameterTypes();
       this.parameterAnnotationsArray = method.getParameterAnnotations();
}
```

再看Builder.build()

![ServiceMethod.Builder.build-1](http://o75vlu0to.bkt.clouddn.com/ServiceMethod.Builder.build-1.png)

![ServiceMethod.Builder.build-2](http://o75vlu0to.bkt.clouddn.com/ServiceMethod.Builder.build-2.png)

第一句代码调用了**ServiceMethod.createCallAdapter()**:

![](http://o75vlu0to.bkt.clouddn.com/ServiceMethod.createCallAdapter.png)

里面又跳到了**Retrofit.callAdapter**中,这个方法的返回值赋值给**ServiceMethod中callAdapter属性**:

```java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
        return nextCallAdapter(null, returnType, annotations);
}
```

callAdapter的类型**CallAdapter<R,T>**,是一个**interface**

```java
public interface CallAdapter<R, T> {
  Type responseType();
  T adapt(Call<R> call);
  abstract class Factory {
      public abstract CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
                                                Retrofit retrofit);
      protected static Type getParameterUpperBound(int index, ParameterizedType type) {
              return Utils.getParameterUpperBound(index, type);
      }
      protected static Class<?> getRawType(Type type) {
              return Utils.getRawType(type);
      }
  }
}
```

第三句代码 ,`createResponseConverter()`

```java
private Converter<ResponseBody, T> createResponseConverter() {
   Annotation[] annotations = method.getAnnotations();
     try {
        return retrofit.responseBodyConverter(responseType, annotations);
      } catch (RuntimeException e) { 
        // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create converter for %s",responseType);
   }
}
```

ServiceMethod其他代码主要用来**解析注解**

再接着看Retrofit.create()中

```java
OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
```

这里构造了一个OkHttpCall,实际上每一个OkHttpCall都对应于一个请求,它主要完成最基础的网络请求,而我们在接口的返回中看到的Call默认情况就是OkHttpCall了,如果我们添加了自定义的callAdapter,那么它就会将OkHttp适配成我们需要的返回值,并返回给我们.

可以看下**Call**这个接口

```java
public interface Call<T> extends Cloneable {
  Response<T> execute() throws IOException;
  void enqueue(Callback<T> callback);
  boolean isExecuted();
  void cancel();
  boolean isCanceled();
  Call<T> clone();
  Request request();
}
```

而OkHttpCall实现了上面的Call接口.

OkHttpCall.execut()

![](http://o75vlu0to.bkt.clouddn.com/OkHttpCall.execute.png)

可以看到里面封装了**okhttp3.Call**,在这个方法中通过okhttp3.Call发起了请求.

`parseResponse(call.execute())`主要完成了由okhttp3.Response向retrofit.Response的转换,同时也处理了对原始返回的解析.





