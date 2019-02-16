#  DRF序列化

> - 请求认证
> - QuerySet进行序列化

## 简单使用

### QuerySet序列化

```python
class UserInfoSer(serializers.Serializer):
	"""
	以userinfo表为例子，如下是我想要显示的字段
	1-username和password都是本表字段，直接展示就可以，调用对应类别为CharField
	2-user_type为用户类型，这在我们的models定义中是一个choice字段，如果直接取IntegerField的话
	那么返回的将是一个id而已，而不是我们想要的typeid对应的中文名称，因此这一块要指定源source。
	orm内部针对choice字段存在一个get_Foo_display的方法，会返回对应的choice对应的值，那么对应到这里就是
	要显示user_type字段对应的choice的选项值。
	3-外键字段，这里还可以填写对应的外键字段，比如group，对应的就是group对象，.title，就是调用group对象
	的title字段的值，同样也要指定source。
	4-而针对多对多字典在这里指定方法的话其实就不是那么方便了，因此采用一个新的类SerializerMethodField
	，这个类可以允许我们自定义方法来指定显示内容，假如字段名叫rls，那么就可以命名一个get_rls的方法，内部
	操作也很简单，就是拿到所有的role对象以后然后把对象的id和title取出来添加到ret里，返回的ret即是rls的值
	"""
    username = serializers.CharField()
    password = serializers.CharField()
    # choices字段
    ut = serializers.CharField(source='get_user_type_display')
    # 外键字段内部是根据.做split，取值，因此可以一直.下去。
    gp = serializers.CharField(source='group.title')
    # 多对多关系，用SerializerMethod进行自定义方法显示，上线的choice，gp都可以用这种方法的形式
    rls = serializers.SerializerMethodField()

    def get_rls(self, row):

        role_obj_list = row.roles.all()
        ret = []
        for item in role_obj_list:
            ret.append({'id': item.id, 'title': item.title})
        return ret
    
 # 对应的CBV
class UserInfo(APIView):

    authentication_classes = []
    permission_classes = []
    throttle_classes = []

    def get(self, request, *args, **kwargs):

        users = models.UserInfo.objects.all()
        ser = UserInfoSer(instance=users, many=True)
        # ser.data返回的是一个OrderedDIct是一个有序字典，实质上还是一个字典
        ret = json.dumps(ser.data, ensure_ascii=False)
        return HttpResponse(ret)
```

访问对应的路由得到的结果如下：

```python
[{"username": "lamber", "password": "123", "ut": "普通用户", "gp": "A组", "rls": [{"id": 1, "title": "医生"}, {"id": 2, "title": "学生"}]}, {"username": "maxiaoyu", "password": "123", "ut": "VIP", "gp": "b组", "rls": [{"id": 2, "title": "学生"}, {"id": 3, "title": "老师"}]}, {"username": "qimaosen", "password": "123", "ut": "SVIP", "gp": "A组", "rls": [{"id": 3, "title": "老师"}]}]
```

其实我们只做了两个事情：

1. 定义一个序列化的类，写上对应的字段
2. 在CBV中实例化序列类，生成序列化后的数据，返回给前端。

在序列化类中我们单独指定了每一个字段的显示，其实这个还可以更简单的的去操作，只要继承内置的ModelSerializer类即可，因此上面的内容就可以改成如下的形式：

```python
class UserInfoSer(serializers.ModelSerializer):

    class Meta:
        model = models.UserInfo
        fields = "__all__"
```

当fields设置为`__all__`的时候，会自动帮你显示所有的字段，这样就不需要自己一个一个手写了，不过也有一个问题就是这里的处理都是最简单化的处理，即对应的group也只是给你显示组id，choices也是显示choice的id而不是显示choice的名称，因此部分字段还是需要我们自己来进行处理，来进行一下改进如下：

```python
class UserInfoSer(serializers.ModelSerializer):

    ut = serializers.CharField(source='get_user_type_display')
    gp = serializers.CharField(source='group.title')
    rls = serializers.SerializerMethodField()

    class Meta:
        model = models.UserInfo
        fields = ['id', 'username', 'ut', 'rls', 'gp']

    def get_rls(self, row):

        role_obj_list = row.roles.all()
        ret = []
        for item in role_obj_list:
            ret.append({'id': item.id, 'title': item.title})
        return ret
```

不过还有一种更为简单的写法，指定一个depth，depth表示深度，我们在外键进行联表查询的时候可以通过“.”一直多级去获取，即一直“.”下去，这里depth就是指定要深入走多少层，：

```python
class UserInfoSer(serializers.ModelSerializer):

    class Meta:
        model = models.UserInfo
        fields = "__all__"
        # 默认ModelSerializer使用主键作为关联字段，但是我们可以使用depth来简单的生成嵌套表示，
        # depth应该是整数，表明嵌套的层级数量，建议3~4层就差不多了，再多就会慢了。
        depth = 1
```

除此之外，我们还可以自定义字段显示，定义一个自定制的Field的类，在内部重写to_representtation方法，方法返回什么就显示什么。

```python
class MyField(serializers.CharField):

    def to_representation(self, value):
        """
        可以自定义这个方法，这个方法返回什么，那么返回的页面数据就是什么，value是从数据库取到的值，
        这个方法一般是用不上，因为我们自定义方法一般就能解决这个问题了。
        """
        return time.time()
    
class UserInfoSer(serializers.ModelSerializer):

    ut = serializers.CharField(source='get_user_type_display')
    gp = serializers.CharField(source='group.title')
    x1 = MyField(source='username')

    class Meta:
        model = models.UserInfo
        fields = ['id', 'username', 'ut', 'gp', 'x1']
```

对应的结果如下：

```python
[{"id": 1, "username": "lamber", "ut": "普通用户", "gp": "A组", "x1": 1550142281.049991}, {"id": 2, "username": "maxiaoyu", "ut": "VIP", "gp": "b组", "x1": 1550142281.052642}, {"id": 3, "username": "qimaosen", "ut": "SVIP", "gp": "A组", "x1": 1550142281.054938}]
```

### Hypermedia API

Hypermedia API，RESTful API最好做到Hypermedia，即返回结果中提供链接，连向其他API方法，使得用户不查文档，也知道下一步应该做什么。以用户组为例：

```python
class UserInfoSer(serializers.ModelSerializer):
	
    """
    这里的lookup_url_kwarg是指的我去反向生成url的时候，参数写的值是什么，
    如果是pk那就是pk，如果是xxx那匹配到的就是xxx，这个是针对URL上有参数的情况
    lookup_field用于执行各个模型实例的对象查找的模型字段。默认为 'pk'。
    请注意，使用 hyperlinked API 时，如果需要使用自定义值，
    则需要确保 API 视图和序列化类设置了 lookup field，不设置默认按照pk来，数据可能是不对的。
    生成反向URL地址，帮助接口人调用的时候可以了解到连接的内容。
    """
    group = serializers.HyperlinkedIdentityField(view_name='gp', lookup_url_kwarg='pk', lookup_field='group_id')

    class Meta:
        model = models.UserInfo
        fields = ['id', 'username', 'group']


class UserInfo(APIView):

    authentication_classes = []
    permission_classes = []
    throttle_classes = []

    def get(self, request, *args, **kwargs):

        users = models.UserInfo.objects.all()
        ser = UserInfoSer(instance=users, many=True)
        # 返回的ser.data是一个OrderedDIct是一个有序字典，实质上还是一个字典
        ret = json.dumps(ser.data, ensure_ascii=False)
        return HttpResponse(ret)
    
class GroupSer(serializers.ModelSerializer):
    
    group = serializers.HyperlinkedIdentityField(view_name='gp', lookup_url_kwarg='pk', lookup_field='id')

    class Meta:
        model = models.UserGroup
        fields = "__all__"


class Group(APIView):

    authentication_classes = []
    permission_classes = []
    throttle_classes = []

    def get(self, request, *args, **kwargs):

        pk = kwargs.get('pk')
        obj = models.UserGroup.objects.filter(id=pk).first()
        # 但凡你要是使用HyperlinkedIdentityField了，你就要加上context这个参数，这样就能获得反向的url
        # 否则会报如下异常：
        # `HyperlinkedIdentityField` requires the request in the serializer context. Add `context={'request': request}` when instantiating the serializer.
        ser = GroupSer(instance=obj, many=False, context={'request': request})
        ret = json.dumps(ser.data)
        return HttpResponse(ret)
```

返回的结果内容如下：

```python
[
    {
        "id": 1,
        "username": "lamber",
        "group": "http://127.0.0.1:8000/group/1"
    },
    {
        "id": 2,
        "username": "maxiaoyu",
        "group": "http://127.0.0.1:8000/group/2"
    },
    {
        "id": 3,
        "username": "qimaosen",
        "group": "http://127.0.0.1:8000/group/1"
    }
]
```

### 请求认证

在请求认证这里其实很类似于Form组件的请求认证，比如我要针对username做一个请求认证

```python
class xxValidator(object):

    def __init__(self, base):
        self.base = base

    def __call__(self, value, *args, **kwargs):
        if not value.startswith(self.base):
            message = '标题必须以%s开头' % self.base
            raise serializers.ValidationError(message)
            
class UserInfoSer(serializers.ModelSerializer):

    username = serializers.CharField(
        # 这里面包含了很多原生的验证规则，比如required，不能为空
        error_messages={
            'required': '不能为空'
        },
        # 但是我们还可以自己去定义验证规则，方法就是定一个类就可以了，比如上面的xxValidator类
        # 如果满足的话不作操作，如果不满足raise一个ValidationError
        # 这里调用了__call__方法，因此类可以直接加括号调用，默认就是调用的__call__方法。
        validators=[xxValidator('呵呵哒'), ],
    )

    group = serializers.HyperlinkedIdentityField(view_name='gp', lookup_url_kwarg='pk', lookup_field='group_id')

    class Meta:
        model = models.UserInfo
        fields = ['id', 'username', 'group']
        
 class UserInfo(APIView):

    authentication_classes = []
    permission_classes = []
    throttle_classes = []

    def post(self, request, *args, **kwargs):
        ser = UserInfoSer(data=request.data)
        if ser.is_valid():
            print(ser.validated_data)
        else:
            print(ser.errors)

        return HttpResponse('测试的post请求')
```

如果post请求过来的内容不是以“呵呵哒”开头的话，ser.errors就会打印出对应的报错

```python
{'username': [ErrorDetail(string='标题必须以呵呵哒开头', code='invalid')]}
```

当然一般情况下我们不会去这么做，Form组件中有钩子，这里同样也有。

```python
def is_valid(self, raise_exception=False):
	
    """省略一部分代码"""
    # 如果没有_validated_data属性的话，那么就会去调用run_validation方法。
    if not hasattr(self, '_validated_data'):
        try:
            self._validated_data = self.run_validation(self.initial_data)
        except ValidationError as exc:
            self._validated_data = {}
            self._errors = exc.detail
        else:
            self._errors = {}

    if self._errors and raise_exception:
        raise ValidationError(self.errors)

    return not bool(self._errors)

def run_validation(self, data=empty):
    """
    Validate a simple representation and return the internal value.

    The provided data may be `empty` if no representation was included
    in the input.

    May raise `SkipField` if the field should not be included in the
    validated data.
    """
    (is_empty_value, data) = self.validate_empty_values(data)
    if is_empty_value:
        return data
    value = self.to_internal_value(data)
    self.run_validators(value)
    return value

def validate_empty_values(self, data):
    """
    Validate empty values, and either:

    * Raise `ValidationError`, 指出无效的数据
    * Raise `SkipField`, 指出这个字段应该被忽略
    * Return (True, data), indicating an empty value that should be
      returned without any further validation being applied.
    * Return (False, data), indicating a non-empty value, that should
      have validation applied as normal.
    """
    if self.read_only:
        return (True, self.get_default())

    if data is empty:
        if getattr(self.root, 'partial', False):
            raise SkipField()
        if self.required:
            self.fail('required')
        return (True, self.get_default())

    if data is None:
        if not self.allow_null:
            self.fail('null')
        return (True, None)

    return (False, data)
```



## 源码流程

我们写一个要做序列化的时候，假如说继承了serializers.Serializer类，我们在CBV中要进行调用：

```python
# 也就是实例化，那么整个代码流程就从实例化开始看
ser = UserInfoSer(instance=users, many=True)
```

实例化的时候会优先指定的两个方法一个是`__new__`另外一个就是构造方法`__init__`。但是我们在Serializer中并没有发现，所以到它集成的父类BaseSerializer中去找。

```python
def __init__(self, instance=None, data=empty, **kwargs):
    # 这个就是我们传递的instance，可能是一个，也可能是多个。
    self.instance = instance
    if data is not empty:
        self.initial_data = data
    self.partial = kwargs.pop('partial', False)
    self._context = kwargs.pop('context', {})
    kwargs.pop('many', None)
    super(BaseSerializer, self).__init__(**kwargs)

def __new__(cls, *args, **kwargs):
    # 重写这个类来自动构建ListSerializer类代替，当设置了many=True时候
    # 我们会去把many这个选项pop出来，如果没有那就是False，不走这里的条件
    # 直接去调用父类的__new__方法.
    if kwargs.pop('many', False):
        # 如果说有many，那么调用类方法many_init
        return cls.many_init(*args, **kwargs)
    return super(BaseSerializer, cls).__new__(cls, *args, **kwargs)
```

在Serializer，如果没有给meta设置list_serializer_class属性，那么list_serializer_class属性就是ListSerializer，最后返回的也就是ListSerializer的对象，接下来执行ListSerializer对象的构造方法，否则执行BaseSerializer里面的构造方法（如果我们重写了这个就会调用我们自己的构造方法）。

```python
@classmethod
def many_init(cls, *args, **kwargs):
    """
    This method implements the creation of a `ListSerializer` parent
    class when `many=True` is used. You can customize it if you need to
    control which keyword arguments are passed to the parent, and
    which are passed to the child.

    Note that we're over-cautious in passing most arguments to both parent
    and child classes in order to try to cover the general case. If you're
    overriding this method you'll probably want something much simpler, eg:

    @classmethod
    def many_init(cls, *args, **kwargs):
        kwargs['child'] = cls()
        return CustomListSerializer(*args, **kwargs)
    """
    allow_empty = kwargs.pop('allow_empty', None)
    child_serializer = cls(*args, **kwargs)
    list_kwargs = {
        'child': child_serializer,
    }
    if allow_empty is not None:
        list_kwargs['allow_empty'] = allow_empty
    list_kwargs.update({
        key: value for key, value in kwargs.items()
        if key in LIST_SERIALIZER_KWARGS
    })
    meta = getattr(cls, 'Meta', None)
    list_serializer_class = getattr(meta, 'list_serializer_class', ListSerializer)
    return list_serializer_class(*args, **list_kwargs)
```

然后我们调用了这个这个对象的data属性：

```python
ret = json.dumps(ser.data, ensure_ascii=False)
```

来看一下这个data方法的实现

```python
@property
def data(self):
    ret = super(Serializer, self).data
    return ReturnDict(ret, serializer=self)

@property
def data(self):
    if hasattr(self, 'initial_data') and not hasattr(self, '_validated_data'):
        msg = (
            'When a serializer is passed a `data` keyword argument you '
            'must call `.is_valid()` before attempting to access the '
            'serialized `.data` representation.\n'
            'You should either call `.is_valid()` first, '
            'or access `.initial_data` instead.'
        )
        raise AssertionError(msg)

    if not hasattr(self, '_data'):
        if self.instance is not None and not getattr(self, '_errors', None):
            self._data = self.to_representation(self.instance)
        elif hasattr(self, '_validated_data') and not getattr(self, '_errors', None):
            self._data = self.to_representation(self.validated_data)
        else:
            self._data = self.get_initial()
    return self._data

def to_representation(self, instance):
    raise NotImplementedError('`to_representation()` must be implemented.')
    
    
# Serializer类的类方法    
def to_representation(self, instance):
    """
    Object instance -> Dict of primitive datatypes.
    """
    ret = OrderedDict()
    fields = self._readable_fields
	
    # fields其实就是咱们定义的这些字段
    for field in fields:
        try:
            # 调用字段的get_attribute方法，并把instance作为参数传递进去。
            attribute = field.get_attribute(instance)
        except SkipField:
            continue

        # We skip `to_representation` for `None` values so that fields do
        # not have to explicitly deal with that case.
        #
        # For related fields with `use_pk_only_optimization` we need to
        # resolve the pk value.
        check_for_none = attribute.pk if isinstance(attribute, PKOnlyObject) else attribute
        if check_for_none is None:
            ret[field.field_name] = None
        else:
            ret[field.field_name] = field.to_representation(attribute)

    return ret
```

找一个Charfield看看

```python
# from rest_framework.serializers import CharField，发现CharField中并没有get_attribute这个方法，所以继续找，它的父类Field发现了这个方法
def get_attribute(self, instance):
    """
    Given the *outgoing* object instance, return the primitive value
    that should be used for this field.
    """
    try:
        return get_attribute(instance, self.source_attrs)
    except (KeyError, AttributeError) as exc:
        if self.default is not empty:
            return self.get_default()
        if self.allow_null:
            return None
        if not self.required:
            raise SkipField()
        msg = (
            'Got {exc_type} when attempting to get a value for field '
            '`{field}` on serializer `{serializer}`.\nThe serializer '
            'field might be named incorrectly and not match '
            'any attribute or key on the `{instance}` instance.\n'
            'Original exception text was: {exc}.'.format(
                exc_type=type(exc).__name__,
                field=self.field_name,
                serializer=self.parent.__class__.__name__,
                instance=instance.__class__.__name__,
                exc=exc
            )
        )
        raise type(exc)(msg)

# source       
if self.source == '*':
    self.source_attrs = []
else:
    # 比如group.title，分隔后就是group，title
    self.source_attrs = self.source.split('.')
    
def get_attribute(instance, attrs):
    """
    Similar to Python's built in `getattr(instance, attr)`,
    but takes a list of nested attributes, instead of a single attribute.

    Also accepts either attribute lookup on objects or dictionary lookups.
    """
    for attr in attrs:
        try:
            if isinstance(instance, collections.Mapping):
                instance = instance[attr]
            else:
                instance = getattr(instance, attr)
        except ObjectDoesNotExist:
            return None
        if is_simple_callable(instance):
            try:
                instance = instance()
            except (AttributeError, KeyError) as exc:
                # If we raised an Attribute or KeyError here it'd get treated
                # as an omitted field in `Field.get_attribute()`. Instead we
                # raise a ValueError to ensure the exception is not masked.
                raise ValueError('Exception raised in callable attribute "{0}"; original exception was: {1}'.format(attr, exc))

    return instance


def is_simple_callable(obj):
    """
    True if the object is a callable that takes no arguments.
    """
    if not (inspect.isfunction(obj) or inspect.ismethod(obj) or isinstance(obj, functools.partial)):
        return False

    sig = inspect.signature(obj)
    params = sig.parameters.values()
    return all(
        param.kind == param.VAR_POSITIONAL or
        param.kind == param.VAR_KEYWORD or
        param.default != param.empty
        for param in params
    )

def isfunction(object):
    """Return true if the object is a user-defined function.

    Function objects provide these attributes:
        __doc__         documentation string
        __name__        name with which this function was defined
        __code__        code object containing compiled function bytecode
        __defaults__    tuple of any default values for arguments
        __globals__     global namespace in which this function was defined
        __annotations__ dict of parameter annotations
        __kwdefaults__  dict of keyword only parameters with defaults"""
    # 和callable是等价的
    return isinstance(object, types.FunctionType)

def ismethod(object):
    """Return true if the object is an instance method.

    Instance method objects provide these attributes:
        __doc__         documentation string
        __name__        name with which this method was defined
        __func__        function object containing implementation of method
        __self__        instance to which this method is bound"""
    return isinstance(object, types.MethodType)
```





## 请求数据校验

