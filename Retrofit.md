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

