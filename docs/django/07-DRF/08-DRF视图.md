



可以单独继承mixin中的create，retrieve，destroy等类，如果写的是特别复杂的就不推荐使用这些了，推荐使用APIVIEW或者使用ModelViewSet，ModelViewSet比较APIVIEW有一个好处就是

如果只想完成简单的CURD，就可以使用ModelViewSet，如果要完成复杂的逻辑，就直接用Generic或者直接使用APIView。



```
has_object_permission
Gerneric_api_view .API_VIEW.get_object,check_object_perssion
```







## Route





## 渲染器

format=json/admin/form，主要就用json还有一个browser的。



允许匿名用户两分钟访问10次，注册用户两分钟访问20次。CORS跨域

