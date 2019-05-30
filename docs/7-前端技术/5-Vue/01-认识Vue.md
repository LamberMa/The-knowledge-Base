# 认识Vue

## Vue的引入方式

- 直接引用(引入文件，或者使用cdn的方式通过src去引入)
```html
# 在生产环境直接引入的时候，需要指明vue的版本，这样可以避免由于版本不一样造成的问题
<script src="https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js"></script>
```

- npm的方式

```html
# 安装vue的最新稳定版
npm install vue
```

## Vue的第一个示例

这里采用的是直接引入vue文件的cdn地址的方式来实现Vue的引入。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <!--将来new的vue实例会控制这个元素中的所有内容-->
    <div id="app">
        <p>{{msg}}</p>
        <p>{{msg + msg}}</p>
        <p>{{ '我是一个简单的测试' }}</p>
        <p>{{ 1 < 2 ? "对的" : '错的' }}</p>
    </div>
    <!--当导入包以后，在浏览器的内存中，就多了一个Vue的构造函数-->
    <script src="https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js"></script>
    <script>
        var app = new Vue({
            // element，表示new这个vue实例要控制页面上的哪一块区域，这个就是要控制app这块区域
            el: '#app', 
            data: {
                // data属性中存放的就是el中用到的数据，前端Vue之类的框架，不提倡手动操作DOM元素
                // 通过Vue提供的指令，可以很方便的把指令渲染到页面上，无需手动操作DOM元素
                msg: 'Hello World', 
            }
        })
    </script>
</body>
</html>
```

1. 首先定义一个div，id为app(圈一块地，用来做我想做的操作)
2. 实例化一个Vue实例，下面的属性在vue对象内部会被加上一个`$`符号，因此在调用的时候也需要加上`$`符号，比如`app.$el`，`app.$data.msg`等
   - el：element，就是指的我们要实际控制页面中的哪一块区域，这里el的值对应的是一个选择器，比如id为app，那么这里的值就应该是"#app"，而且这里只能是id选择器，因为圈的地是唯一的，如果你要圈多快地，那么你就要new多个Vue实例去实现
   - data：要在el对象中用到的数据，通过Vue提供的指令，可以很方便的把内容渲染到页面上，不需要手动去操作DOM元素，同时Vue也不建议去手动操作DOM元素。
3. 使用插值表达式将数据展示出来，其实类似于Django的模板语法，双大括号的形式，插值语法支持变量以及简单的运算，比如两个变量相加，或者1+1，又或者直接显示一个简单的字符串也是可以的，亦或是三目运算符也是支持的，但是不建议在插值表达式中体现过多的逻辑，因为不好维护，而且麻烦。

