# DRF访问限制

> 接口需要有访问的限制，而不是随意的无限制的访问，DRF提供了访问限制方面的内容。

## 控制访问频率

> 首先来看针对匿名用户的访问限制的设计思路，首先说访问限制不是绝对的，针对非登录的匿名用户来讲，IP可以说是用户唯一的标识，那么我们可以针对访问的源IP来做限制，但是用户的IP是可以换的，比如用户更换代理，那么可能的IP就是无限多个，但是访问人是同一个，因此没有办法做到绝对的限制。
>
> 同样的，针对登录用户也是不能做到绝对的完全的限制，现在基本每个账号都会绑定手机号，但是你拦不住用户直接去某宝买手机号，代理注册，所以说这些限制在应用层面上来讲，只能说是一定程度做了限制。

匿名用户访问频率的设计思路：

- 首先捕获用户的IP地址，这个ip地址可以在request的META信息中去取到，remote_addr或者是x_forward_for

- 准备一个字典，这个字典可以是一个全局级别的变量，或者是数据库亦或是redis。

- 捕获到的IP作为key，而value为一个列表去存储用户的访问记录

  ```python
  - 访问记录的实体为一个时间戳，用户每访问一次向value列表中的首部添加一个
  - 当用户来访问的时候首先来看这个全局字典中有没有对应的IP的key，如果没有的话那么就添加，并向列表中追加当前的时间戳。
  - 如果有当前的key证明这个ip的匿名用户之前有访问过，那么就从后往前逐个对比时间差值是否超过设置的时间间隔，如果有超过的就POP掉，这就是为啥记录时间戳的时候要记录在前面，因为在清理过期访问记录的时候可以使用pop方便的去删除记录。
  - 去除掉过期记录以后去排查当前列表的长度，比如一分钟内只让访问10次，那么假如当前的列表长度只有小于10的时候才允许你进行下一次访问。
  ```
那么具体的代码实现逻辑可以看下面的简单代码
```python
"""所有导入的包都省略了"""

def md5(user):

    ctime = time.time()
    m = hashlib.md5(bytes(user, encoding='utf8'))
    m.update(bytes(str(ctime), encoding='utf8'))

    return m.hexdigest()


# 全局变量可以，数据库可以，redis也可以
VISIT_RECORD = {}


class VisitThrottle(BaseThrottle):

    def __init__(self):

        self.history = None

    def allow_request(self, request, view):
        """
        如果return true表示可以继续访问，如果返回false，表示访问频率太高，被限制访问
        :param request:
        :param view:
        :return:
        """
        # 1、获取用户IP，我们知道说这里的request是已经被封装过的
        # 之前看封装的request的时候，里面有一个getattr方法，当我们调用META的时候，封装的
        # request中并不存在这个属性，因此会去原生的request去找，因此仍然可以直接调用。
        # 因此这里可以省略_request，直接调用META
        remote_addr = request.META.get('REMOTE_ADDR')
        current_time = time.time()
		
        # 如果捕获到的IP地址没有再这个访问记录的字典里的话，那么就增加一下。
        if remote_addr not in VISIT_RECORD:
            VISIT_RECORD[remote_addr] = [current_time, ]
            return True
        
        # 如果说有访问记录，那么就把访问记录的列表赋值给self.history
        self.history = VISIT_RECORD.get(remote_addr)
        
        # 清理过期的访问记录，比如我现在允许你一分钟你访问10次，也就是60s内访问10次
        # 那么新的一次访问进来以后，首先比对新的时间戳和最老的时间戳，如果差值大于60s
        # 那么证明这个访问记录可能已经很久远了，就可以删掉了，以此类推，依次对比。
        while self.history and self.history[-1] < (current_time - 10):
            self.history.pop()
		
        # 将老旧的信息删掉以后，然后看看列表的长度，一分钟内10次的话那么列表长度就应该小于等于10
        # 当10的时候此时就已经达到访问上限了，因此只有当列表长度小于10的才允许继续访问插值。
        if len(self.history) < 3:
            # 时间越近的塞到列表的首部
            self.history.insert(0, current_time)
            return True

    def wait(self):
        """提示你距离访问还需要多久"""
        return time.time() - self.history[-1]

class AuthView(APIView):

    authentication_classes = []
    permission_classes = []
    throttle_classes = [VisitThrottle, ]

    def post(self, request, *args, **kwargs):

        ret = {'errmsg': None, 'errcode': 0}
        try:
            user = request._request.POST.get('username')
            pwd = request._request.POST.get('password')

            user_obj = models.UserInfo.objects.filter(username=user, password=pwd).first()
            if not user_obj:
                ret['errcode'] = 1000
                ret['errmsg'] = '用户名或密码错误'
                return JsonResponse(ret)

            token = md5(user)
            models.UserToken.objects.update_or_create(user=user_obj, defaults={'token': token })
            ret['token'] = token
            ret['errmsg'] = 'ok'

        except Exception as e:
            ret['errcode'] = 1002
            ret['errmsg'] = '请求异常'
            ret['token'] = ''

        return JsonResponse(ret)
```

经过这样一个流程就可以实现我们的访问频次的限制了，接下来看一下这部分源码流程是如何实现的。

## 源码流程

> 其实仔细观察可以发现这一块的逻辑和认证以及权限的逻辑是非常相像的，入口依然是dispatch。

```python
def initial(self, request, *args, **kwargs):
	
    """此处省略很多代码"""

    self.perform_authentication(request)
    self.check_permissions(request)
    
    # 找到这里，节流的相关设置
    self.check_throttles(request)
```

看一下check_throttles都干了什么

```python
# get_throttles的逻辑和前面的认证和权限的管理都是一个套路，这里不多做赘述了
# 这里依然是返回了self中的throtle_classes的类的对象的列表，如果没有的话
# 会找到父类APIView中的如下这一段内容，如果要全局化直接写到settings中即可。
# throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES
def get_throttles(self):
    """
    Instantiates and returns the list of throttles that this view uses.
    """
    return [throttle() for throttle in self.throttle_classes]

def check_throttles(self, request):
    """
    Check if request should be throttled.
    Raises an appropriate exception if the request is throttled.
    """
    for throttle in self.get_throttles():
        # 调用对象的allow_request方法，如果返回true表明可以继续访问，如果不反回或者返回False
        # 表示不能够继续访问。当不能够继续访问的时候会进入到下面的条件中，指定throttled方法
        # 调用throttled方法的同时还调用了throttle的wait方法，也就是我们自定义的wait方法
        if not throttle.allow_request(request, self):
            self.throttled(request, throttle.wait())
```

throttled方法直接抛出了异常，这里接收的wait就是我们返回的秒数。

```python
def throttled(self, request, wait):
    """
    If request is throttled, determine what kind of exception to raise.
    """
    raise exceptions.Throttled(wait)

# 看一下这个异常类的调用
class Throttled(APIException):
    status_code = status.HTTP_429_TOO_MANY_REQUESTS
    default_detail = _('Request was throttled.')
    extra_detail_singular = 'Expected available in {wait} second.'
    extra_detail_plural = 'Expected available in {wait} seconds.'
    default_code = 'throttled'

    def __init__(self, wait=None, detail=None, code=None):
        if detail is None:
            detail = force_text(self.default_detail)
        if wait is not None:
            wait = math.ceil(wait)
            detail = ' '.join((
                detail,
                force_text(ungettext(self.extra_detail_singular.format(wait=wait),
                                     self.extra_detail_plural.format(wait=wait),
                                     wait))))
        self.wait = wait
        super(Throttled, self).__init__(detail, code)
```

## 内置的节流类

> 其实上面这一堆操作内置的节流类就已经帮我们实现了

首先我们自己定义的节流类都应该继承BaseThrottle。

```python
class BaseThrottle(object):
    """
    Rate throttling of requests.
    """

    def allow_request(self, request, view):
        """
        Return `True` if the request should be allowed, `False` otherwise.
        """
        raise NotImplementedError('.allow_request() must be overridden')

    def get_ident(self, request):
        """
        Identify the machine making the request by parsing HTTP_X_FORWARDED_FOR
        if present and number of proxies is > 0. If not use all of
        HTTP_X_FORWARDED_FOR if it is available, if not use REMOTE_ADDR.
        注意这里已经帮我们实现了获取IP的逻辑，因此我们不需要自己从META中去取了，直接调用
        self.get_indent就可以拿到匿名用户的地址标识了。
        """
        xff = request.META.get('HTTP_X_FORWARDED_FOR')
        remote_addr = request.META.get('REMOTE_ADDR')
        num_proxies = api_settings.NUM_PROXIES

        if num_proxies is not None:
            if num_proxies == 0 or xff is None:
                return remote_addr
            addrs = xff.split(',')
            client_addr = addrs[-min(num_proxies, len(addrs))]
            return client_addr.strip()

        return ''.join(xff.split()) if xff else remote_addr

    def wait(self):
        """
        Optionally, return a recommended number of seconds to wait before
        the next request.
        """
        return None
```

我们上面写到了一定时间内限制用户的访问频次，其实DRF内部已经以一个简单的限流类帮我们写好了，具体内容的实现如下：

```python
class SimpleRateThrottle(BaseThrottle):
    """
    A simple cache implementation, that only requires `.get_cache_key()`
    to be overridden.

    The rate (requests / seconds) is set by a `rate` attribute on the View
    class.  The attribute is a string of the form 'number_of_requests/period'.

    Period should be one of: ('s', 'sec', 'm', 'min', 'h', 'hour', 'd', 'day')

    Previous request information used for throttling is stored in the cache.
    """
    cache = default_cache
    timer = time.time
    cache_format = 'throttle_%(scope)s_%(ident)s'
    scope = None
    # 4、从这里可以知道这个key是从settings中取，因此我们需要在REST_FRAMEWORK中配置这个key
    # 对应的value是一个字典，对应的有一个scope的key，比如我在我自己定义的类里配置scope为test_throttle
    # 那么它就会去配置文件里找key为scope的内容，所以说我们定义的这个scope其实就是配置文件里对应的key
    # 那么这个key对应的value应该填写什么呢？应该填写类似于3/m这样类似的内容，其中m代表分钟，s秒，h小时
    # d代表day，3/m就表示每分钟3次的意思，其他的类推。那么最后我们拿到的这个THROTTLE_RATES其实就是3/m
    THROTTLE_RATES = api_settings.DEFAULT_THROTTLE_RATES
	
    # 1、实现构造方法，在对象内部找rate这个属性，如果rate属性不存在这回调用self.get_rate
    def __init__(self):
        if not getattr(self, 'rate', None):
            # 5、回到这里，selt.rate其实就是3/m了。
            self.rate = self.get_rate()
        # 6、调用parse_rate对我们的3/m进行数据处理，接收到请求数和秒数。
        self.num_requests, self.duration = self.parse_rate(self.rate)

    def get_cache_key(self, request, view):
        """
        Should return a unique cache-key which can be used for throttling.
        Must be overridden.

        May return `None` if the request should not be throttled.
        """
        raise NotImplementedError('.get_cache_key() must be overridden')
	
    # 2、当rate不存在的时候调用当前方法，方法内部去调用scope属性，当没有设置scope的时候会抛出异常
    def get_rate(self):
        """
        Determine the string representation of the allowed request rate.
        """
        if not getattr(self, 'scope', None):
            msg = ("You must set either `.scope` or `.rate` for '%s' throttle" %
                   self.__class__.__name__)
            raise ImproperlyConfigured(msg)

        try:
            # 3、如果设置了scope那么就会去self.THROTTLE_RATES中找对应的key，这个值是一个字典
            return self.THROTTLE_RATES[self.scope]
        except KeyError:
            msg = "No default throttle rate set for '%s' scope" % self.scope
            raise ImproperlyConfigured(msg)
	
    # 7、对self.rate进行数据处理
    def parse_rate(self, rate):
        """
        Given the request rate string, return a two tuple of:
        <allowed number of requests>, <period of time in seconds>
        """
        if rate is None:
            return (None, None)
        # 8、我们的模式都是x/x的形式，所以这里直接用/给隔开了，获取到的num就是次数，period就是间隔。
        num, period = rate.split('/')
        # 9、int转换成整形
        num_requests = int(num)
        # 10、将对应的日期转换成秒，period可能为s，m，h，d，取第一个，所以你写day，hour，minute，second
        # 其实也没有什么关系，因为都是取的首字母，下面的操作相当于取对应的字典的key得到对应的秒数
        duration = {'s': 1, 'm': 60, 'h': 3600, 'd': 86400}[period[0]]
        # 11、返回请求数和秒数
        return (num_requests, duration)
	
    # 12、请求进来以后在源码流程中会调用allow_request方法。
    def allow_request(self, request, view):
        """
        Implement the check to see if the request should be throttled.

        On success calls `throttle_success`.
        On failure calls `throttle_failure`.
        """
        if self.rate is None:
            return True
		
        # 13、这里直接调用self.get_cache_key，如果仔细看上面的源码的话会发现调用原生的会直接抛异常
        # 报错信息告知我们必须自己重写去覆盖原来的方法，其实原来的方法就是个框子。
        # 我们最上面针对匿名用户是以匿名用户的ip作为key的，所以这里我们直接在我们的限流类里调用
        # self.get_ident返回ip赋值给self.key。
        self.key = self.get_cache_key(request, view)
        if self.key is None:
            return True
		
        # 14、从缓存中拿到所有的记录，就相当于之前的VISIT_RECORD，这个缓存是Django内置的缓存
        self.history = self.cache.get(self.key, [])
        # 15、获取当前时间
        self.now = self.timer()

        # 16、这一块的逻辑基本就和我们之前写的是一样的了。
        while self.history and self.history[-1] <= self.now - self.duration:
            self.history.pop()
        if len(self.history) >= self.num_requests:
            # 17、如果说列表长度大于等于请求数那么证明已经到了访问上线了，直接return False
            return self.throttle_failure()
        # 18、否则调用throttle_success插入时间戳到列表头部，并返回True表示不限流。
        return self.throttle_success()

    def throttle_success(self):
        """
        Inserts the current request's timestamp along with the key
        into the cache.
        """
        self.history.insert(0, self.now)
        self.cache.set(self.key, self.history, self.duration)
        return True

    def throttle_failure(self):
        """
        Called when a request to the API has failed due to throttling.
        """
        return False

    def wait(self):
        """
        Returns the recommended next request time in seconds.
        """
        if self.history:
            remaining_duration = self.duration - (self.now - self.history[-1])
        else:
            remaining_duration = self.duration

        available_requests = self.num_requests - len(self.history) + 1
        if available_requests <= 0:
            return None

        return remaining_duration / float(available_requests)
```

那么我们自己的限流类其实相对来讲就简单很多了，只需要继承如上面这个类再做简单的配置即可：

```python
from rest_framework.throttling import SimpleRateThrottle

# 最后我们需要做的内容就简简单单的这一点就可以了。
class VisitThrottle(SimpleRateThrottle):

    scope = 'test_throttle'

    def get_cache_key(self, request, view):
        # 如果是针对登录用户我们可以返回用户的唯一登录名，或者用户的pk(id)
        return self.get_ident(request)
```

对应的settings配置如下:

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': ['api.utils.auth.Authtication', ],
    'UNAUTHENTICATED_USER': None,
    'UNAUTHENTICATED_TOKEN': None,
    'DEFAULT_PERMISSION_CLASSES': ['api.utils.permission.MyPermission'],
    'DEFAULT_THROTTLE_CLASSES': [''],
    'DEFAULT_THROTTLE_RATES': {
        'test_throttle': '10/m'
    }
}
```

其他的提供内置限流类如下以供参考：

```python
# 限制匿名用户
class AnonRateThrottle(SimpleRateThrottle):
    """
    Limits the rate of API calls that may be made by a anonymous users.

    The IP address of the request will be used as the unique cache key.
    """
    scope = 'anon'

    def get_cache_key(self, request, view):
        if request.user.is_authenticated:
            return None  # Only throttle unauthenticated requests.

        return self.cache_format % {
            'scope': self.scope,
            'ident': self.get_ident(request)
        }


# 限制登录用户的
class UserRateThrottle(SimpleRateThrottle):
    """
    Limits the rate of API calls that may be made by a given user.

    The user id will be used as a unique cache key if the user is
    authenticated.  For anonymous requests, the IP address of the request will
    be used.
    """
    scope = 'user'

    def get_cache_key(self, request, view):
        if request.user.is_authenticated:
            ident = request.user.pk
        else:
            ident = self.get_ident(request)

        return self.cache_format % {
            'scope': self.scope,
            'ident': ident
        }


# 这个是应用在局部视图上的，但是我们应用到局部视图直接在对应的视图类写throttle_classes就可以了
# 因此这个基本用不到。
class ScopedRateThrottle(SimpleRateThrottle):
    """
    Limits the rate of API calls by different amounts for various parts of
    the API.  Any view that has the `throttle_scope` property set will be
    throttled.  The unique cache key will be generated by concatenating the
    user id of the request, and the scope of the view being accessed.
    """
    scope_attr = 'throttle_scope'

    def __init__(self):
        # Override the usual SimpleRateThrottle, because we can't determine
        # the rate until called by the view.
        pass

    def allow_request(self, request, view):
        # We can only determine the scope once we're called by the view.
        self.scope = getattr(view, self.scope_attr, None)

        # If a view does not have a `throttle_scope` always allow the request
        if not self.scope:
            return True

        # Determine the allowed request rate as we normally would during
        # the `__init__` call.
        self.rate = self.get_rate()
        self.num_requests, self.duration = self.parse_rate(self.rate)

        # We can now proceed as normal.
        return super(ScopedRateThrottle, self).allow_request(request, view)

    def get_cache_key(self, request, view):
        """
        If `view.throttle_scope` is not set, don't apply this throttle.

        Otherwise generate the unique cache key by concatenating the user id
        with the '.throttle_scope` property of the view.
        """
        if request.user.is_authenticated:
            ident = request.user.pk
        else:
            ident = self.get_ident(request)

        return self.cache_format % {
            'scope': self.scope,
            'ident': ident
        }
```

## 小结

依然需要注意，配置文件配置的节流类是全局的，是给所有的人用的，如果说单独需要其他的方案的需要单独处理，直接在对应的视图CBV中写`throttle_classes = []`单独处理就可以了。