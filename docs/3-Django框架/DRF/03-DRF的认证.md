# Django Rest FrameWork

## DRF的认证

### 简单环境准备

model class简单点，一个用来存储用户信息一个用来存储token认证信息。

```python
from django.db import models

class UserInfo(models.Model):
    user_type_choices = (
        (1, '普通用户'),
        (2, 'VIP'),
        (3, 'SVIP'),
    )
    user_type = models.IntegerField(choices=user_type_choices)
    username = models.CharField(max_length=64, unique=True)
    password = models.CharField(max_length=64)

class UserToken(models.Model):
    # 还可以在这里加token时间，最多使用次数，比如可以设置一个expire_time作为过期时间
    # 避免每次访问的时候都刷新token
    user = models.OneToOneField('UserInfo', on_delete=models.CASCADE)
    token = models.CharField(max_length=64)
```

### 认证源码解析

> 仍然以dispatch作为切入口，查看request封装过程的时候有对认证部分的封装

这里只着重认证部分，其他的部分会回过头来重新说明的。

```python
# initialize_request
def initialize_request(self, request, *args, **kwargs):
    """
    返回初始化的request对象，这个对象是drf的Request的对象，相当于在原基础上封装了更多的内容
    但是原来的django的request给封装到了返回的对象里，所以说依然可以使用之前的request。
    """
    parser_context = self.get_parser_context(request)
    
    return Request(
        request,
        parsers=self.get_parsers(),
        # self.authentication_classes中列出的类的对象
        authenticators=self.get_authenticators(),
        negotiator=self.get_content_negotiator(),
        parser_context=parser_context
    )
```

在初始化封装request的过程中，返回了一个DRF的Request对象，里面封装了原生的request以及一个authenticators，这个authenticators是调用`self.get_authenticators()`获得的。

来看一下get_authenticators是如何操作的。

```python
# 遍历self.authentications_classes返回对应每一个item的实例，填充到列表里。因此我们自己定义的CBV中应该需要存在一个列表authentication_classes，列表里是我们自定义的认证类
def get_authenticators(self):
    """
    Instantiates and returns the list of authenticators that this view can use.
    """
    return [auth() for auth in self.authentication_classes]
```

此时的这个self是谁？这个self现在是我们定义的CBV，因此去找self.authentication_classes的时候会优先到我们自己的类里面去找，假如我在我自己的类里面定义了这个内容：

```python
class OrderView(APIView):
    
    # 表示在你写的这个类中应用drf的认证规则
    authentication_classes = [Authtication, ]
    
    def get(self, request, *args, **kwargs):
		pass

    def post(self, request, *args, **kwargs):
        pass
```

那么get_authenticators返回的就是我们自己定义的authentication_classes中的对象的列表。到此为止我们在认证方面知道dispatch在DRF中被初始化，request被重新封装，而且封装后的内容包含原生的request，以及一个我们自定制的认证规则类的对象列表。

接下来继续看dispatch的内容，在dispatch的后续代码中调用了initial方法。

```python
def dispatch(self, request, *args, **kwargs):

    self.args = args
    self.kwargs = kwargs
    request = self.initialize_request(request, *args, **kwargs)
    self.request = request
    self.headers = self.default_response_headers  # deprecate?

    try:
        # 调用initial方法
        self.initial(request, *args, **kwargs)

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

找到self.initial方法

```python
def initial(self, request, *args, **kwargs):
    """
    Runs anything that needs to occur prior to calling the method handler.
    """
    self.format_kwarg = self.get_format_suffix(**kwargs)

    # Perform content negotiation and store the accepted info on the request
    neg = self.perform_content_negotiation(request)
    request.accepted_renderer, request.accepted_media_type = neg

    # Determine the API version, if versioning is in use.
    version, scheme = self.determine_version(request, *args, **kwargs)
    request.version, request.versioning_scheme = version, scheme

    # 确认进来的请求是否被允许。
    self.perform_authentication(request)
    self.check_permissions(request)
    self.check_throttles(request)
```

看最后的self.perform_authentication方法，这个方法是用来确认这个进来的请求是否经过允许，也就是进来的这个请求是否认证成功？

```python
def perform_authentication(self, request):
    """
    Perform authentication on the incoming request.

    Note that if you override this and simply 'pass', then authentication
    will instead be performed lazily, the first time either
    `request.user` or `request.auth` is accessed.
    """
    request.user
```

返回了一个request对象的user方法，注意此时的request是已经被封装过的request，我们去DRF的Request类中去找这么一个方法，之所以没有加括号是因为被property修饰过了：

```python
@property
def user(self):
    """
    Returns the user associated with the current request, as authenticated
    by the authentication classes provided to the request.
    """
    # 如果当前对象中(注意self指的是CBV的对象)没有_user这个属性的话那么就调用_authenticate方法
    # 如果有直接返回self._user
    if not hasattr(self, '_user'):
        with wrap_attributeerrors():
            self._authenticate()
    return self._user
```

我们看，如果说认证成功以后会返回self.\_user，如果没有_user则会调用self.\_authentication()方法

```python
# 来看一下_authenticate干了点什么
def _authenticate(self):
    """
    Attempt to authenticate the request using each authentication instance
    in turn.
    """
    # 循环self.authenticators中的对象
    for authenticator in self.authenticators:
        try:
            # 执行认证类对象的authenticate方法，如果没有异常的话，返回一个用户认证的元组
            # 所以我们在写authenticate方法的时候会返回一个元组
            user_auth_tuple = authenticator.authenticate(self)
        except exceptions.APIException:
            # 如果捕获到异常了，那么就调用_not_authenticated方法。
            self._not_authenticated()
            # 向上级抛出异常
            raise
        
        if user_auth_tuple is not None:
            # 如果有返回值，那么证明执行过相关认证赋值操作
            # 如果返回None的话，那么循环继续，表示当前认证不处理，交给下一个认证类去处理。
            # 赋值，request.user和request.auth就是这么来的。因此有返回值必须是一个元组
            # 元组里面必须有两个元素，第一个给request.user，第二个给request.auth
            # 对应的request.user就是user对象，request.auth就是token对象。
            self._authenticator = authenticator
            self.user, self.auth = user_auth_tuple
            return
	
    # 也有可能都不处理，返回的都是None，这个时候就默认赋值
    self._not_authenticated()
```

我们在\_authenticate中发现，循环遍历了我们封装的self.authenticators中的对象，然后调用对象的authenticate方法，也就是说在我们自己写的类里面要有一个authenticate方法，这个方法应该返回什么？上面的方法支持三种返回值：

1. 自己返回元组：元组中包含认证用户的user对象，以及认证结果的认证对象（比如token表中的对象）。
2. 抛出异常：调用`self._not_authenticated()`，抛出的异常会在\_authenticate中捕获到。
3. 返回None，也就是所有的认证规则全部通过，但是没有返回值，此时调用`self._not_authenticated()`.

来看看`self._not_authenticated()`都干了点什么？

```python
def _not_authenticated(self):
	# 设置默认的_authenticator为空，
    self._authenticator = None
	# 如果设置了UNAUTHENTICATED_USER那么就调用（AnonymousUser），否则就返回None
    if api_settings.UNAUTHENTICATED_USER:
        self.user = api_settings.UNAUTHENTICATED_USER()
    else:
        self.user = None
	# 如果设置了UNAUTHENTICATED_TOKEN那么就调用，否则就返回None
    if api_settings.UNAUTHENTICATED_TOKEN:
        self.auth = api_settings.UNAUTHENTICATED_TOKEN()
    else:
        self.auth = None
```

其实就是为默认的用户做一个设置，给通过认证却没有返回具体认证信息的人一个身份（匿名用户）。到此为止，认证是ok了。不过刚才走的authenticate_classes是我们自己定义的，难道我们每写一个类都要自己这样定义一下么？其实不是的，针对这个认证规则是有一个全局设置的。

之前的authenticate_classes是默认优先找自己定义的cbv中的这个属性，假如说没有的话那么就应该去父类去找了，那么父类中是如何定义的？最后我们发现这个内置的authentication_classess，它是一个api_settings的一个配置项：

```python
class APIView(View):

    # 下面的这些策略可以设置成全局的，也可以为每一个单独的CBV去设置。
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES
    parser_classes = api_settings.DEFAULT_PARSER_CLASSES
    authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
    ………………………………
    
# 这个配置是从配置文件中获取的。
api_settings = APISettings(None, DEFAULTS, IMPORT_STRINGS)

def reload_api_settings(*args, **kwargs):
    setting = kwargs['setting']
    # 会去配置文件中找REST_FRAMEWORK这样一个key，我们就可以把配置项写到这里。
    if setting == 'REST_FRAMEWORK':
        api_settings.reload()
```

默认这个变量是没有东西的，需要我们自己去加，加的方法如下：

```python
# settings.py文件最后进行添加
REST_FRAMEWORK = {
    # 这里写的是认证类的路径，可以单独扔到一个py文件里
    'DEFAULT_AUTHENTICATION_CLASSES': ['api.utils.auth.FirstAuth', 'api.utils.xxx.xxx'],
    # 推荐使用None，这个玩意是可以调用的，因此也可以写一个lambda表达式返回我们自定义的信息就可以了。
    'UNAUTHENTICATED_USER': None, # request.user = None
    'UNAUTHENTICATED_TOKEN': None, # request.auth = None
}
```

认证类不要和view写到一起，这样视图函数就只有视图相关的逻辑，而认证相关的被剥离到auth.py中了。因此在api项目下新建一个utils的目录，新建一个auth.py将我们的认证逻辑都扔到auth.py里面去，这里的值我们知道对应的是一个列表，里面写的都是类的全路径（类似middleware那种写法）。

这样我们实际的每一个业务类就不用写这些内容了，相当于全局添加了认证，但是也有例外的页面，比如认证页面，认证页面是不需要添加认证机制的，你得先通过了认证页面拿到了token访问别的需要认证的页面的时候才需要认证，你现在都没登录，我认证页面还加了认证，那就没办法访问了。因此针对一些业务类需要放开这个权限，放开的方法也很简单，定义一个空列表就行了。因为在类的内部定义了，因此会优先走类内部的，实现了单独的类的特殊放开。

```python
class AuthView(APIView):
    
    # 直接跳过验证
    authentication_classes = []

    def post(self, request, *args, **kwargs):
 		………………

    def get(self, request, *args, **kwargs):
        ………………
```

### 内置认证类

```python
from rest_framework.authentication import BaseAuthentication
```

为了规范都要继承默认的BaseAuthentication类。authenticate_header是认证失败（这个是浏览器的一种认证机制）的时候给你的浏览器返回的响应头。BaseAuthentication这个类其实啥都没写，就是定义了两个方法，一个authenticate一个authenticate_header方法。而且authenticate必须要重写，否则会抛出异常。

其他的内置认证类型还包括Session的，Token的，RemoteUser的，不过这些认证都是基于Django的，以Session为例，它会去获取request.user这个选项，但是如果我们自己去写session的时候是没有这个内容的。request.user是基于Django的。再比如RemoteUser是基于Django的auth去进行认证的。因此内置的这几种方法其实都是有一定局限性的，因此认证类，一般是我们自己去写，不会去用到DRF原生的。

说一下BasicAuthentication，这个是基于浏览器的账号密码认证，在访问页面的时候会以浏览器的形式弹出来一个账号密码的认证输入框，浏览器会把输入的账号密码进行加密扔到请求头，加密的形式如下：

```python
HTTP_AUTHORIZATION: "basic (用户名+密码)base64转码"
```

在BasicAuthentication中会去获取这一部分：

```python
def get_authorization_header(request):
    """
    Return request's 'Authorization:' header, as a bytestring.

    Hide some test client ickyness where the header can be unicode.
    """
    # 从request.META中把这个HTTP_AUTHORIZATION取出来。
    auth = request.META.get('HTTP_AUTHORIZATION', b'')
    if isinstance(auth, text_type):
        # Work around django test client oddness
        auth = auth.encode(HTTP_HEADER_ENCODING)
    return auth
```

拿到base64转码后的内容进行解码的操作。

```python
class BasicAuthentication(BaseAuthentication):
    """
    HTTP Basic authentication against username/password.
    """
    www_authenticate_realm = 'api'

    def authenticate(self, request):
        """
        Returns a `User` if a correct username and password have been supplied
        using HTTP Basic authentication.  Otherwise returns `None`.
        """
        auth = get_authorization_header(request).split()
		# 如果返回为None或者不是basic认证，那么return None
        if not auth or auth[0].lower() != b'basic':
            return None
        if len(auth) == 1:
            msg = _('Invalid basic header. No credentials provided.')
            raise exceptions.AuthenticationFailed(msg)
        elif len(auth) > 2:
            msg = _('Invalid basic header. Credentials string should not contain spaces.')
            raise exceptions.AuthenticationFailed(msg)

        try:
            # 开始base64解码，注意split和partition类似，split不会取分隔符，但是partition会取
            auth_parts = base64.b64decode(auth[1]).decode(HTTP_HEADER_ENCODING).partition(':')
        except (TypeError, UnicodeDecodeError, binascii.Error):
            msg = _('Invalid basic header. Credentials not correctly base64 encoded.')
            raise exceptions.AuthenticationFailed(msg)

        userid, password = auth_parts[0], auth_parts[2]
        return self.authenticate_credentials(userid, password, request)
```

## 小结

### authenticate返回值

- None：我不管了，下一个认证来执行
- 抛出异常，`raise exception.AuthenticationFailed('Auth Failed')`
- 返回一个元组，(user obj，auth obj)，因此元素一赋值给request.user，元素二赋值给request.auth

### 使用范围

- 局部使用：视图类中写一个静态字段，authentication_classes，列表里面是类的名称，这里和全局不一样。

- 全局使用，在settings中配置：

  ```python
  REST_FRAMEWORK = {
      # 这里写的是认证类的路径，可以单独扔到一个py文件里
      'DEFAULT_AUTHENTION_CLASSES': ['api.utils.auth.Authtication', ],
      'UNAUTHENTICATED_USER': None,
      'UNAUTHENTICATED_TOKEN': None,
  }
  ```
