# Request的封装

> 在Django的RestFrameWork中，我们常用的是CBV（注意不是不能用FBV，FBV可以通过装饰器的形式去实现），而且CBV继承的不是原生的django的View，而是DRF在Django的view基础上进一步封装的APIView，在这一步进行了Request的重新封装，接下来看一下新的Request都封装了哪些内容。

## 封装过程

整个流程从一个请求进来开始说：

```python
from rest_framework.views import APIView

class AuthView(APIView):
    pass
```

rest_framework的APIView默认继承Django的View，CBV中找到对应的request的请求方法是利用反射来实现的，原生Django View通过dispatch方法来实现：

```python
# 原生Django View
def dispatch(self, request, *args, **kwargs):
    # Try to dispatch to the right method; if a method doesn't exist,
    # defer to the error handler. Also defer to the error handler if the
    # request method isn't on the approved list.
    if request.method.lower() in self.http_method_names:
        handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
    else:
        handler = self.http_method_not_allowed
    return handler(request, *args, **kwargs)
```

不过我们知道，我们可以在自己的CBV中重写这个dispatch，借此来实现更多的操作。先调用一下父类（View）的dispatch方法，然后在调用之前和调用之后就可以做一些我们自己的操作了，其实DRF的APIView也是这么干的。

```python
# DRF的dispatch，drf的dispatch和django的dispatch几乎是一致的，只不过在原来的基础上加了一些钩子，也就是我们所说的在调用父类的操作之前，执行一些其他的方法。这些钩子分别存在于开始，结束以及异常的时候。
def dispatch(self, request, *args, **kwargs):

    self.args = args
    self.kwargs = kwargs
    # 1、首先从这里开始，DRF对django的request做了封装。
    # 这个request已经发生变化了，是经过drf加工过后的request了。
    request = self.initialize_request(request, *args, **kwargs)
    self.request = request
    self.headers = self.default_response_headers  # deprecate?

    try:
        self.initial(request, *args, **kwargs)

        # Get the appropriate handler method
        if request.method.lower() in self.http_method_names:
            handler = getattr(self, request.method.lower(),
                              self.http_method_not_allowed)
        else:
            handler = self.http_method_not_allowed

        response = handler(request, *args, **kwargs)

    except Exception as exc:
        response = self.handle_exception(exc)

    self.response = self.finalize_response(request, response, *args, **kwargs)
    return self.response
```

在dispatch这一步骤，request被重新赋值，已经不是原生的request了，而是经过DRF封装之后的Request。调用了初始化的方法，来看一下这个方法具体做了什么：

```python
# initialize_request
def initialize_request(self, request, *args, **kwargs):
    """
    返回初始化的request对象，这个对象是drf的Request的对象，相当于在原基础上封装了更多的内容
    但是原来的django的request给封装到了返回的对象里，所以说依然可以使用之前的request。
    """
    parser_context = self.get_parser_context(request)
    
    # _request：原生的request，authenticators获取认证类的对象。
    return Request(
        request, # 这里返回的Request的实例对象，将原生的Django的request封装进去了。
        parsers=self.get_parsers(),
        authenticators=self.get_authenticators(),
        negotiator=self.get_content_negotiator(),
        parser_context=parser_context
    )
```

在初始化的这一步里返回了Request实例化后的对象并复制给request，到此为止，Request被封装完毕，接下来大概粗略的浏览一下我们可以调用这个封装的request的什么属性。

## 常用属性

- \_request： 在构造函数中，`self._request = request`，原生的request被封装进\_request中
- content_type：返回请求头的类型，默认从`self._request.META`中去取。
- query_params：GET请求的参数，如果我们自己拿的话需要`request._request.GET.get`，其实这一个步骤相当于DRF帮我去做了。

值得注意的就是request内部还有一个`__getattr__`方法，这个方法允许我们调用封装的request中没有的属性的时候，会去原生的request中去调用返回。

```python
def __getattr__(self, attr):
    """
    If an attribute does not exist on this instance, then we also attempt
    to proxy it to the underlying HttpRequest object.
    """
    try:
        return getattr(self._request, attr)
    except AttributeError:
        return self.__getattribute__(attr)
```

