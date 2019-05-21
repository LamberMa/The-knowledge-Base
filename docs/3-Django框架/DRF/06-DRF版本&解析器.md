# DRF版本&解析器

> - 编辑日期：2019-2-13
>
> 思考一个问题：restframework你都继承过哪些类

## DRF版本

> 版本可以放在URL里，也可以放在请求头，只要是可以获取到就行。

```python
def initial(self, request, *args, **kwargs):
    """
    Runs anything that needs to occur prior to calling the method handler.
    """
    self.format_kwarg = self.get_format_suffix(**kwargs)

    neg = self.perform_content_negotiation(request)
    request.accepted_renderer, request.accepted_media_type = neg

    # Determine the API version, if versioning is in use.
    # 调用self.determine_version方法，会收到两个返回值
    # 两个返回值分别对应的第一个是determine_version的返回值，另外一个就是这个version类对象。
    version, scheme = self.determine_version(request, *args, **kwargs)
    # 对应的版本号和version类对象会被赋值给request.version和request.versioning_scheme
    request.version, request.versioning_scheme = version, scheme

    # Ensure that the incoming request is permitted
    self.perform_authentication(request)
    self.check_permissions(request)
    self.check_throttles(request)
    
    
def determine_version(self, request, *args, **kwargs):
    """
    If versioning is being used, then determine any API version for the
    incoming request. Returns a two-tuple of (version, versioning_scheme)
    """
    if self.versioning_class is None:
        # 如果没有设置的话也就是说根本不用，那就返回None，None的元组
        return (None, None)
    # versioning_class = api_settings.DEFAULT_VERSIONING_CLASS
    scheme = self.versioning_class()
    # 返回这个类的对象。调用对象的determine_version方法
    return (scheme.determine_version(request, *args, **kwargs), scheme)
```

注意这个在类中声明的时候，这个可不是类列表，class后面没有es，这个就是单独的一个类。

```python
class UserView(APIView):

    authentication_classes = []
    permission_classes = []
    throttle_classes = []

    versioning_class = ParamVersion

    def get(self, request, *args, **kwargs):
        return HttpResponse('userlist')
```

### 以参数形式获取

不过在版本这一块，不用自己写，drf中已经提供了这一块的内容。

```python
# 直接调用内置的QueryParameterVersioning就可以了。
from rest_framework.versioning import QueryParameterVersioning

class UserView(APIView):

    authentication_classes = []
    permission_classes = []
    throttle_classes = []
    versioning_class = [QueryParameterVersioning, ]

    def get(self, request, *args, **kwargs):
        return HttpResponse('当前的版本为： "%s"' % request.version)
```

来看一下这个版本类的实现：

```python
class QueryParameterVersioning(BaseVersioning):
    """
    GET /something/?version=0.1 HTTP/1.1
    Host: example.com
    Accept: application/json
    """
    invalid_version_message = _('Invalid version in query parameter.')

    def determine_version(self, request, *args, **kwargs):
        # 而且这里如果获取不到配置的VERSION_PARAM还有一个默认的version配置：DEFAULT_VERSION
        # version_param就是你传递的那个version的key是什么，后面的就是你不传默认是什么
        # 这些都需要在配置文件中去体现。
        version = request.query_params.get(self.version_param, self.default_version)
        if not self.is_allowed_version(version):
            raise exceptions.NotFound(self.invalid_version_message)
        return version
	
    # reverse用来反向生成url，viewname就是我们的那个name，而且version我们不用加
    # 因为request.version携带在request里面了，因此我们可以省略这个参数。
    def reverse(self, viewname, args=None, kwargs=None, request=None, format=None, **extra):
        url = super(QueryParameterVersioning, self).reverse(
            viewname, args, kwargs, request, format, **extra
        )
        if request.version is not None:
            return replace_query_param(url, self.version_param, request.version)
        return url
    
# 设置都允许哪些version
# allowed_versions = api_settings.ALLOWED_VERSIONS
def is_allowed_version(self, version):
    if not self.allowed_versions:
        return True
    return ((version is not None and version == self.default_version) or
            (version in self.allowed_versions))
```

### 在路径中体现

> 上面的version参数传递是以参数的形式传递的，但是实际情况下这种使用的其实并不多，更多的其实是将version包含在路径中，比如xxx.xxx.com/api/v1这样的形式。当然DRF关于这种情况也给你写好了。

```python
class URLPathVersioning(BaseVersioning):
    """
    To the client this is the same style as `NamespaceVersioning`.
    The difference is in the backend - this implementation uses
    Django's URL keyword arguments to determine the version.

    An example URL conf for two views that accept two different versions.

    urlpatterns = [
        url(r'^(?P<version>[v1|v2]+)/users/$', users_list, name='users-list'),
        url(r'^(?P<version>[v1|v2]+)/users/(?P<pk>[0-9]+)/$', users_detail, name='users-detail')
    ]

    GET /1.0/something/ HTTP/1.1
    Host: example.com
    Accept: application/json
    """
    invalid_version_message = _('Invalid version in URL path.')

    def determine_version(self, request, *args, **kwargs):
        version = kwargs.get(self.version_param, self.default_version)
        if version is None:
            version = self.default_version

        if not self.is_allowed_version(version):
            raise exceptions.NotFound(self.invalid_version_message)
        return version

    def reverse(self, viewname, args=None, kwargs=None, request=None, format=None, **extra):
        if request.version is not None:
            kwargs = {} if (kwargs is None) else kwargs
            kwargs[self.version_param] = request.version

        return super(URLPathVersioning, self).reverse(
            viewname, args, kwargs, request, format, **extra
        )
```

更推荐使用这种配置，其他的内置方案仅供参考：

```python
class NamespaceVersioning(BaseVersioning):
    """
    To the client this is the same style as `URLPathVersioning`.
    The difference is in the backend - this implementation uses
    Django's URL namespaces to determine the version.

    An example URL conf that is namespaced into two separate versions

    # users/urls.py
    urlpatterns = [
        url(r'^/users/$', users_list, name='users-list'),
        url(r'^/users/(?P<pk>[0-9]+)/$', users_detail, name='users-detail')
    ]

    # urls.py
    urlpatterns = [
        url(r'^v1/', include('users.urls', namespace='v1')),
        url(r'^v2/', include('users.urls', namespace='v2'))
    ]

    GET /1.0/something/ HTTP/1.1
    Host: example.com
    Accept: application/json
    """
    invalid_version_message = _('Invalid version in URL path. Does not match any version namespace.')

    def determine_version(self, request, *args, **kwargs):
        resolver_match = getattr(request, 'resolver_match', None)
        if resolver_match is None or not resolver_match.namespace:
            return self.default_version

        # Allow for possibly nested namespaces.
        possible_versions = resolver_match.namespace.split(':')
        for version in possible_versions:
            if self.is_allowed_version(version):
                return version
        raise exceptions.NotFound(self.invalid_version_message)

    def reverse(self, viewname, args=None, kwargs=None, request=None, format=None, **extra):
        if request.version is not None:
            viewname = self.get_versioned_viewname(viewname, request)
        return super(NamespaceVersioning, self).reverse(
            viewname, args, kwargs, request, format, **extra
        )

    def get_versioned_viewname(self, viewname, request):
        return request.version + ':' + viewname


class HostNameVersioning(BaseVersioning):
    """
    GET /something/ HTTP/1.1
    Host: v1.example.com
    Accept: application/json
    """
    hostname_regex = re.compile(r'^([a-zA-Z0-9]+)\.[a-zA-Z0-9]+\.[a-zA-Z0-9]+$')
    invalid_version_message = _('Invalid version in hostname.')

    def determine_version(self, request, *args, **kwargs):
        hostname, separator, port = request.get_host().partition(':')
        match = self.hostname_regex.match(hostname)
        if not match:
            return self.default_version
        version = match.group(1)
        if not self.is_allowed_version(version):
            raise exceptions.NotFound(self.invalid_version_message)
        return version

    # We don't need to implement `reverse`, as the hostname will already be
    # preserved as part of the REST framework `reverse` implementation.
    
 
class AcceptHeaderVersioning(BaseVersioning):
    """
    GET /something/ HTTP/1.1
    Host: example.com
    Accept: application/json; version=1.0
    """
    invalid_version_message = _('Invalid version in "Accept" header.')

    def determine_version(self, request, *args, **kwargs):
        media_type = _MediaType(request.accepted_media_type)
        version = media_type.params.get(self.version_param, self.default_version)
        version = unicode_http_header(version)
        if not self.is_allowed_version(version):
            raise exceptions.NotAcceptable(self.invalid_version_message)
        return version

    # We don't need to implement `reverse`, as the versioning is based
    # on the `Accept` header, not on the request URL.
```

## DRF解析器

```python
def _load_post_and_files(self):
    """Populate self._post and self._files if the content-type is a form type"""
    if self.method != 'POST':
        self._post, self._files = QueryDict(encoding=self._encoding), MultiValueDict()
        return
    if self._read_started and not hasattr(self, '_body'):
        self._mark_post_parse_error()
        return

    if self.content_type == 'multipart/form-data':
        if hasattr(self, '_body'):
            # Use already read data
            data = BytesIO(self._body)
        else:
            data = self
        try:
            self._post, self._files = self.parse_file_upload(self.META, data)
        except MultiPartParserError:
            # An error occurred while parsing POST data. Since when
            # formatting the error the request handler might access
            # self.POST, set self._post and self._file to prevent
            # attempts to parse POST data again.
            # Mark that an error occurred. This allows self.__repr__ to
            # be explicit about it instead of simply representing an
            # empty POST
            self._mark_post_parse_error()
            raise
    # 如果是这个头才去request.body中去解析数据，不仅如此，还对数据格式有要求
    # 数据格式要求必须是a=1&b=2&c=3这种的。form表单的提交默认就是这个头，数据格式默认也是这样的
    # 当然Ajax也是可以提交的，虽然你data写的是字典，不过内部也会给你转成上面的数据格式，而且ajax还可以
    # 定制请求头headers。
    elif self.content_type == 'application/x-www-form-urlencoded':
        self._post, self._files = QueryDict(self.body, encoding=self._encoding), MultiValueDict()
    else:
        self._post, self._files = QueryDict(encoding=self._encoding), MultiValueDict()
```

还有一种情况也是request.body有值，但是request.post没有值

```javascript
# 这个时候不仅头不满足，数据格式也不满足，都不满足。所有request.post依然没有值。
$.ajax({
    url: xxx,
    method: xxx,
    data: JSON.Stringfy({'a':1,'b':2})
})
```

上面这个是Django的，接下来看DRF解析器

1. 获取用户请求
2. 获取用户请求体
3. 根据用户请求头和parser_classes中支持的全给你求头进行比较
4. 符合的处理类进行请求的处理
5. 赋值给request.data



调用request.data的时候生效。



```python
# JSONParser
class JSONParser(BaseParser):
    """
    Parses JSON-serialized data.
    """
    # 在JSONParser中定义了一个media type
    media_type = 'application/json'
    renderer_class = renderers.JSONRenderer
    strict = api_settings.STRICT_JSON

    def parse(self, stream, media_type=None, parser_context=None):
        """
        Parses the incoming bytestream as JSON and returns the resulting data.
        """
        parser_context = parser_context or {}
        encoding = parser_context.get('encoding', settings.DEFAULT_CHARSET)

        try:
            decoded_stream = codecs.getreader(encoding)(stream)
            parse_constant = json.strict_constant if self.strict else None
            return json.load(decoded_stream, parse_constant=parse_constant)
        except ValueError as exc:
            raise ParseError('JSON parse error - %s' % six.text_type(exc))
```





```python
# FormParser
class FormParser(BaseParser):
    """
    Parser for form data.
    """
    media_type = 'application/x-www-form-urlencoded'

    def parse(self, stream, media_type=None, parser_context=None):
        """
        Parses the incoming bytestream as a URL encoded form,
        and returns the resulting QueryDict.
        """
        parser_context = parser_context or {}
        encoding = parser_context.get('encoding', settings.DEFAULT_CHARSET)
        data = QueryDict(stream.read(), encoding=encoding)
        return data
```





```python
# 1、我们知道最后获取到的结果可以从request.data中拿到，但是这个request是经过封装后的Request的对象
# 因此我们直接去Request里面去找data属性，首先调用了_hasattr，注意这里是带下划线的
@property
def data(self):
    # 3、默认的self._full_data为Empty，那么_hasattr会返回False，直接调用_load_data_and_files
    if not _hasattr(self, '_full_data'):
        self._load_data_and_files()
    return self._full_data

# 2、找到下划线_has_attr,它会从当前self中去取对应的name，如果这个name取到了是Empty，那么就返回False
def _hasattr(obj, name):
    return not getattr(obj, name) is Empty

# 4、_full_data默认值
self._full_data = Empty

# 5、Empty的定义
class Empty(object):
    """
    Placeholder for unset attributes.
    Cannot use `None`, as that may be a valid value.
    """
    pass

# 6、_load_data_and_files方法
def _load_data_and_files(self):
    """
    Parses the request content into `self.data`.
    """
    # 默认的self._data = Empty，所以条件满足直接进入调用
    if not _hasattr(self, '_data'):
        # 调用self._parse方法
        self._data, self._files = self._parse()
        if self._files:
            self._full_data = self._data.copy()
            self._full_data.update(self._files)
        else:
            self._full_data = self._data

        # if a form media type, copy data & files refs to the underlying
        # http request so that closable objects are handled appropriately.
        if is_form_media_type(self.content_type):
            self._request._post = self.POST
            self._request._files = self.FILES
            
# 7、self._parse方法
def _parse(self):
    """
    格式化request的文本，并且返回一个两个元素的元组(data，files)
    May raise an `UnsupportedMediaType`, or `ParseError` exception.
    """
    # 8、获取用户的请求头
    media_type = self.content_type
    try:
        # 10、获取用户的request.body的内容，以stream的形式
        stream = self.stream
    except RawPostDataException:
        if not hasattr(self._request, '_post'):
            raise
        # If request.POST has been accessed in middleware, and a method='POST'
        # request was made with 'multipart/form-data', then the request stream
        # will already have been exhausted.
        if self._supports_form_parsing():
            return (self._request.POST, self._request.FILES)
        stream = None

    if stream is None or media_type is None:
        if media_type and is_form_media_type(media_type):
            empty_data = QueryDict('', encoding=self._request._encoding)
        else:
            empty_data = {}
        empty_files = MultiValueDict()
        return (empty_data, empty_files)
	
    # 13、调用parser选择器，后面的self.parsers其实就是支持的对象。parser_classes
    # self中也有content-type，所以这个self是请求对象，包含用户请求的content-type
    parser = self.negotiator.select_parser(self, self.parsers)

    if not parser:
        raise exceptions.UnsupportedMediaType(media_type)

    try:
        parsed = parser.parse(stream, media_type, self.parser_context)
    except Exception:
        # If we get an exception during parsing, fill in empty data and
        # re-raise.  Ensures we don't simply repeat the error when
        # attempting to render the browsable renderer response, or when
        # logging the request or similar.
        self._data = QueryDict('', encoding=self._request._encoding)
        self._files = MultiValueDict()
        self._full_data = self._data
        raise

    # Parser classes may return the raw data, or a
    # DataAndFiles object.  Unpack the result as required.
    try:
        return (parsed.data, parsed.files)
    except AttributeError:
        empty_files = MultiValueDict()
        return (parsed, empty_files)

# 9、加了property属性，可以直接像值一样去使用，这个content_type实际上就是去获取用户实际的请求头。
@property
def content_type(self):
    meta = self._request.META
    return meta.get('CONTENT_TYPE', meta.get('HTTP_CONTENT_TYPE', ''))

# 11、stream
@property
def stream(self):
    """
    Returns an object that may be used to stream the request content.
    """
    # 默认的_stream就是Empty所以，直接调用_load_stream方法
    if not _hasattr(self, '_stream'):
        self._load_stream()
    return self._stream

# 12、返回BytesIO形式的body信息
def _load_stream(self):
    """
    Return the content body of the request, as a stream.
    """
    meta = self._request.META
    try:
        content_length = int(
            meta.get('CONTENT_LENGTH', meta.get('HTTP_CONTENT_LENGTH', 0))
        )
    except (ValueError, TypeError):
        content_length = 0

    if content_length == 0:
        self._stream = None
    elif not self._request._read_started:
        self._stream = self._request
    else:
        self._stream = io.BytesIO(self.body)
    
```



看一切的开头，入口还是dispatch

```python
def initialize_request(self, request, *args, **kwargs):
    """
    Returns the initial request object.
    """
    parser_context = self.get_parser_context(request)

    return Request(
        request,
        # 这样Request的实例就会存在self.parsers这个属性可以调用。
        parsers=self.get_parsers(),
        authenticators=self.get_authenticators(),
        negotiator=self.get_content_negotiator(),
        parser_context=parser_context
    )

def get_parsers(self):
    """
    Instantiates and returns the list of parsers that this view can use.
    """
    return [parser() for parser in self.parser_classes]
```



最后这个解析器的属性写在全局就可以了。





http协议的请求方法

常用的请求头（状态码，方法等）：

- application/





refer用来做防盗链，user_agent，accept，host等。





