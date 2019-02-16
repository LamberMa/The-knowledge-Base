# DRF权限

> DRF的权限是在认证的基础上做进一步的认证，比如用户根据不同的用户类型，他们可能会有不同的权限，比如普通用户，管理员，超级管理员等等。

## 权限的基本使用

DRF的权限的设置流程其实和认证是大同小异的，需要注意的就是权限的使用范围，配置在settings配置文件的就是全局性质的，局部视图的直接在对应的视图类设置`permission_classes`属性。

## 源码流程

> 在源码流程部分其实和认证部分是大同小异的，入口依旧是dispatch。

找到dispatch中封装的initial方法

```python
 def initial(self, request, *args, **kwargs):
	
     """此处省略很多代码~~~~~"""
    
     self.perform_authentication(request)
        
     # 找到这一步骤
     self.check_permissions(request)
     self.check_throttles(request)
```

看看check_permissions做了什么

```python
def check_permissions(self, request):
    """
    Check if the request should be permitted.
    Raises an appropriate exception if the request is not permitted.
    """
    for permission in self.get_permissions():
        if not permission.has_permission(request, self):
            self.permission_denied(
                request, message=getattr(permission, 'message', None)
            )
```

我们发现它又在self（也就是我们CBV的对象里）找这个get_permissions了，如果我们自己没有定义的话那么就会去父类里面找。找到父类的APIView，对应的有这么一条：

```python
class APIView(View):

    ………………
    permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
    ………………
```

这样就和之前的认证部分一致了，如果要自己定义的话记得KEY要写成对应的上面的名字，别写错了，否则是不会生效的。

继续来看check_permissions：

```python
# get_permissions()做了一件事就是遍历self.permission_classes然后通过列表生成式返回所有权限类的对象列表。
def get_permissions(self):
    """
    Instantiates and returns the list of permissions that this view requires.
    """
    return [permission() for permission in self.permission_classes]

# permission_denied中抛出了异常
def permission_denied(self, request, message=None):
    """
    If request is not permitted, determine what kind of exception to raise.
    """
    if request.authenticators and not request.successful_authenticator:
        raise exceptions.NotAuthenticated()
    raise exceptions.PermissionDenied(detail=message)

for permission in self.get_permissions():
    # 调用权限对象的has_permission方法，如果返回的是True，即权限通过则继续循环
    # 如果返回False，则权限不通过，进入if条件中。
    if not permission.has_permission(request, self):
        # 调用self的permission_denied，
        self.permission_denied(
            # messages在permission中取了信息，也就是说我们可以自定义这个message
            request, message=getattr(permission, 'message', None)
        )
```

## 内置权限

```python
@six.add_metaclass(BasePermissionMetaclass)
class BasePermission(object):
    """
    A base class from which all permission classes should inherit.
    """

    def has_permission(self, request, view):
        """
        Return `True` if permission is granted, `False` otherwise.
        """
        return True

    def has_object_permission(self, request, view, obj):
        """
        Return `True` if permission is granted, `False` otherwise.
        """
        return True
```

为了规范，我们写的权限类都要继承BasePermission类，其他的还有一些默认的权限类，这里仅作参考，一般生产环境的时候是不会用到这些内置类的，而是根据业务自己开发权限。

```python
class AllowAny(BasePermission):
    """
    Allow any access.
    This isn't strictly required, since you could use an empty
    permission_classes list, but it's useful because it makes the intention
    more explicit.
    """

    def has_permission(self, request, view):
        return True


class IsAuthenticated(BasePermission):
    """
    Allows access only to authenticated users.
    """

    def has_permission(self, request, view):
        return bool(request.user and request.user.is_authenticated)


class IsAdminUser(BasePermission):
    """
    Allows access only to admin users.
    """

    def has_permission(self, request, view):
        return bool(request.user and request.user.is_staff)


class IsAuthenticatedOrReadOnly(BasePermission):
    """
    The request is authenticated as a user, or is a read-only request.
    """

    def has_permission(self, request, view):
        return bool(
            request.method in SAFE_METHODS or
            request.user and
            request.user.is_authenticated
        )


class DjangoModelPermissions(BasePermission):
    """
    The request is authenticated using `django.contrib.auth` permissions.
    See: https://docs.djangoproject.com/en/dev/topics/auth/#permissions

    It ensures that the user is authenticated, and has the appropriate
    `add`/`change`/`delete` permissions on the model.

    This permission can only be applied against view classes that
    provide a `.queryset` attribute.
    """

    # Map methods into required permission codes.
    # Override this if you need to also provide 'view' permissions,
    # or if you want to provide custom permission codes.
    perms_map = {
        'GET': [],
        'OPTIONS': [],
        'HEAD': [],
        'POST': ['%(app_label)s.add_%(model_name)s'],
        'PUT': ['%(app_label)s.change_%(model_name)s'],
        'PATCH': ['%(app_label)s.change_%(model_name)s'],
        'DELETE': ['%(app_label)s.delete_%(model_name)s'],
    }

    authenticated_users_only = True

    def get_required_permissions(self, method, model_cls):
        """
        Given a model and an HTTP method, return the list of permission
        codes that the user is required to have.
        """
        kwargs = {
            'app_label': model_cls._meta.app_label,
            'model_name': model_cls._meta.model_name
        }

        if method not in self.perms_map:
            raise exceptions.MethodNotAllowed(method)

        return [perm % kwargs for perm in self.perms_map[method]]

    def _queryset(self, view):
        assert hasattr(view, 'get_queryset') \
            or getattr(view, 'queryset', None) is not None, (
            'Cannot apply {} on a view that does not set '
            '`.queryset` or have a `.get_queryset()` method.'
        ).format(self.__class__.__name__)

        if hasattr(view, 'get_queryset'):
            queryset = view.get_queryset()
            assert queryset is not None, (
                '{}.get_queryset() returned None'.format(view.__class__.__name__)
            )
            return queryset
        return view.queryset

    def has_permission(self, request, view):
        # Workaround to ensure DjangoModelPermissions are not applied
        # to the root view when using DefaultRouter.
        if getattr(view, '_ignore_model_permissions', False):
            return True

        if not request.user or (
           not request.user.is_authenticated and self.authenticated_users_only):
            return False

        queryset = self._queryset(view)
        perms = self.get_required_permissions(request.method, queryset.model)

        return request.user.has_perms(perms)


class DjangoModelPermissionsOrAnonReadOnly(DjangoModelPermissions):
    """
    Similar to DjangoModelPermissions, except that anonymous users are
    allowed read-only access.
    """
    authenticated_users_only = False


# 针对对象的认证
class DjangoObjectPermissions(DjangoModelPermissions):
    """
    The request is authenticated using Django's object-level permissions.
    It requires an object-permissions-enabled backend, such as Django Guardian.

    It ensures that the user is authenticated, and has the appropriate
    `add`/`change`/`delete` permissions on the object using .has_perms.

    This permission can only be applied against view classes that
    provide a `.queryset` attribute.
    """
    perms_map = {
        'GET': [],
        'OPTIONS': [],
        'HEAD': [],
        'POST': ['%(app_label)s.add_%(model_name)s'],
        'PUT': ['%(app_label)s.change_%(model_name)s'],
        'PATCH': ['%(app_label)s.change_%(model_name)s'],
        'DELETE': ['%(app_label)s.delete_%(model_name)s'],
    }

    def get_required_object_permissions(self, method, model_cls):
        kwargs = {
            'app_label': model_cls._meta.app_label,
            'model_name': model_cls._meta.model_name
        }

        if method not in self.perms_map:
            raise exceptions.MethodNotAllowed(method)

        return [perm % kwargs for perm in self.perms_map[method]]

    def has_object_permission(self, request, view, obj):
        # authentication checks have already executed via has_permission
        queryset = self._queryset(view)
        model_cls = queryset.model
        user = request.user

        perms = self.get_required_object_permissions(request.method, model_cls)

        if not user.has_perms(perms, obj):
            # If the user does not have permissions we need to determine if
            # they have read permissions to see 403, or not, and simply see
            # a 404 response.

            if request.method in SAFE_METHODS:
                # Read permissions already checked and failed, no need
                # to make another lookup.
                raise Http404

            read_perms = self.get_required_object_permissions('GET', model_cls)
            if not user.has_perms(read_perms, obj):
                raise Http404

            # Has read permissions.
            return False

        return True
```

