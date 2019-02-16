# DRF

> - [武Sir博客](www.cnblogs.com/wupeiqi/articles/7805382.html)

## RestFul

###Restful说明

- REST与技术无关，代表的是一种软件架构风格，REST是Representational State Transfer的简称，中文翻译为“表征状态转移”
- REST从资源的角度类审视整个网络，它将分布在网络中某个节点的资源通过URL进行标识，客户端应用通过URL来获取资源的表征，获得这些表征致使这些应用转变状态
- 所有的数据，不管是通过网络获取的还是操作（增删改查）的数据，都是资源，将一切数据视为资源是REST区别与其他架构风格的最本质属性
- 对于REST这种面向资源的架构风格，有人提出一种全新的结构理念，即：面向资源架构（ROA：Resource Oriented Architecture）

###Restful设计规范

1. API与用户的通信协议，总是使用[HTTPs协议](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)。（推荐使用https）
2. 域名 

```python
- https://api.example.com：尽量将API部署在专用域名（会存在跨域问题，需要自己解决）
- https://example.org/api/：尽量使用这种，这样的API很简单
```

3. 版本：可能存在多个版本共存的情况，版本之间可能也需要过渡，因此url后面一般还会带一个版本

```python
- URL，如：https://api.example.com/v1/    一般版本号放在url
- 版本好也可以加到请求头跨域时，引发发送多次请求
```

4. 路径，视网络上任何东西都是资源（面向资源编程），均使用名词表示（可复数），比如订单order，而不是get_order，或者delete_order这样的url。

```python
- https://api.example.com/v1/zoos
- https://api.example.com/v1/animals
- https://api.example.com/v1/employees
```

5. method

```python
- GET      ：从服务器取出资源（一项或多项）
- POST    ：在服务器新建一个资源
- PUT      ：在服务器更新资源（客户端提供改变后的完整资源，即全部更新）
- PATCH  ：在服务器更新资源（客户端提供改变的属性，即局部更新）
- DELETE ：从服务器删除资源
```

6. 过滤，通过在url上传参的形式传递搜索条件

```python
- https://api.example.com/v1/zoos?limit=10：指定返回记录的数量
- https://api.example.com/v1/zoos?offset=10：指定返回记录的开始位置
- https://api.example.com/v1/zoos?page=2&per_page=100：指定第几页，以及每页的记录数
- https://api.example.com/v1/zoos?sortby=name&order=asc：指定返回结果按照哪个属性排序，以及排序顺序
- https://api.example.com/v1/zoos?animal_type_id=1：指定筛选条件
```

7. 状态码：根据状态码给用户做提示，但是仅仅用状态码是不够的，因此大多使用code+状态码结合使用。status code可以在HTTPResponse中以参数形式返回。现在主要还是以code为主。有的对状态码有需求，有的没有需求，在写接口前要问清楚。

```html
200 OK - [GET]：服务器成功返回用户请求的数据，该操作是幂等的（Idempotent）。
201 CREATED - [POST/PUT/PATCH]：用户新建或修改数据成功。
202 Accepted - [*]：表示一个请求已经进入后台排队（异步任务）
204 NO CONTENT - [DELETE]：用户删除数据成功。
400 INVALID REQUEST - [POST/PUT/PATCH]：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的。
401 Unauthorized - [*]：表示用户没有权限（令牌、用户名、密码错误）。
403 Forbidden - [*] 表示用户得到授权（与401错误相对），但是访问是被禁止的。
404 NOT FOUND - [*]：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的。
406 Not Acceptable - [GET]：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
410 Gone -[GET]：用户请求的资源被永久删除，且不会再得到的。
422 Unprocesable entity - [POST/PUT/PATCH] 当创建一个对象时，发生一个验证错误。
500 INTERNAL SERVER ERROR - [*]：服务器发生错误，用户将无法判断发出的请求是否成功。

更多看这里：http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html
```

8. 错误处理，状态码是4xx时，应返回错误信息，推荐用error当做key。

```json
{
    error: "Invalid API key"
}
```

9. 返回结果，针对不同操作，服务器向用户返回的结果应该符合以下规范。

```
GET /collection：返回资源对象的列表（数组）
GET /collection/resource：返回单个资源对象
POST /collection：返回新生成的资源对象
PUT /collection/resource：返回完整的资源对象
PATCH /collection/resource：返回完整的资源对象
DELETE /collection/resource：返回一个空文档
```

10. Hypermedia API，RESTful API最好做到Hypermedia，即返回结果中提供链接，连向其他API方法，使得用户不查文档，也知道下一步应该做什么。（其实就是为了让你省事，再给你返回一个url）

```
{
    "link": {
        "rel":   "collection https://www.example.com/zoos",
        "href":  "https://api.example.com/zoos",
        "title": "List of zoos",
        "type":  "application/vnd.yourformat+json"
     }
}
```

**谈谈你对Restful规范的认识**

```
本质上就是一个规范，让后台更容易处理，让前端更容易记住这些url，在url上能体现更多的操作。
restful有些适用项目，有些也不适用，不一定要完全符合这个标准，最重要的还是契合项目。
协同开发的时候共同遵循这个规范，让操作更加统一。
推荐使用CBV的方式
```

## DRF的安装

```shell
# 安装过程极其简单
pip3 install djangorestframework
```