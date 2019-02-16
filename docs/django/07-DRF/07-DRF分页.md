# 分页

>- 简单分页：看第n页，每页显示m条数据
>- 指定分页：在第n个位置，向后查看n条数据。
>
>- 加密分页：只能看上一页下一页





```python
class GenericViewSet(ViewSetMixin, generics.GenericAPIView):
    """
    The GenericViewSet class does not provide any actions by default,
    but does include the base set of generic view behavior, such as
    the `get_object` and `get_queryset` methods.
    """
    pass
```



