# 指令系统

## 常见指令

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
    <style>
        [v-cloak]{
            display: none;
        }
    </style>
</head>
<body>
    <div id="app">
        <!--v-cloak可以解决渲染的时候模板插值表达式一闪而过的问题
            在没有渲染完成的时候是这个p标签是display为none的，渲染完成以后会移除v-cloak属性-->
        <p v-cloak>{{msg1}}</p>
        <!--v-text和插值表达式功能差不多，但是有点区别，默认v-text没有闪烁问题。v-text会覆盖标签内的其他内容，但是插值表达式只是替换占位符，不会替换标签内的其他内容-->
        <h4 v-text="msg1"></h4>
        <!--v-html可以帮我们将html的文本信息解析，否则输出的就是str形式的html文本，注意v-html也会覆盖标签内的内容-->
        <h1>{{msg2}}</h1>
        <h1 v-html="msg2"></h1>
        <!--v-bind是vue中提供的一个绑定属性的指令，简写方式可以直接写一个冒号。-->
        <!--v-bind会把等号后面的内容当成是一个js代码去执行，下面的mytitle就会看做是js的一个变量-->
        <!--v-bind只能实现数据的单向绑定，从Model绑定到View中，不能通过View绑定到Model中。-->
        <input type="text" v-bind:placeholder="mytitle"><br />
        <!--如果是一个表达式的话也是没有问题的-->
        <input type="text" :placeholder="mytitle + '123'">
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el: '#app', // 这里只能是id，因为作用的领域只能有一个
            data: {
                msg1: 'msg1',
                msg2: '<img src="../medias/image/centos.png" />',
                mytitle: 'this is a custom title'
            }
        })
    </script>
</body>
</html>
```

### v-cloak

在使用插值表达式的时候，可能会出现一个问题，就是在由模板语法替换到最后的数据的时候能看到渲染的过程，虽然过程很短，一闪而过，但是体验仍然不是最好的。v-cloak可以解决这个问题，就是为这个属性添加一个display为none的样式，当渲染完成以后该属性会被移除

### v-text

v-text也是插值表达式，但是不存在闪烁的问题，它是直接覆盖标签内的其他内容，只替换模板占位符，但是不替换其他的内容。

### v-html

v-html可以将我们的替换的字符串同时做html语义的解析，比如如果v-text，一个a标签就是字符串形式的a标签，如果使用v-html，这个a标签将会被解析。

### v-bind

提供绑定属性的指令，比如input标签中我们知道有placeholder属性，那么就可以使用v-bind进行绑定属性

```html
<input type="text" v-bind:placeholder="mytitle"><br />
```

当然v-bind你可以省略，v-bind直接省略出来一个冒号就可以，比如下面的

```html
<input type="text" :placeholder="mytitle + '123'">
```

而且需要注意的是，这里的mytitle是data中的一个属性，它会被认为是一个js的变量，一个data中的key。同时v-bind只能实现从model层到view层的绑定，不能实现View层到model层的绑定。其它常见的用法还有设置标签的class类属性等等。针对class类的绑定，v-bind后面的值是一个object，key是class的名称，value是属性值，这个属性值会去data里面找，如果为True，那么这个class会被加再进来。

**示例1**

```html
<body>
    <div id="app">
        <!--比如绑定title属性，鼠标悬浮会显示出来-->
        <h3 v-bind:title='title'>呵呵哒</h3>
        <!--可以直接使用:的方式省略v-bind，这是一种简便的写法-->
        <img :src="imgSrc" alt="">
        <!--针对class类的绑定，v-bind后面的值是一个对象，其中key box2是类名，isGreen是属性值-->
        <!--属性值会去data对象里去找isGreen这个属性，如果是true，那么box2会被附加到class内部-->
        <!--你能看到的就是两个class，一个box1，一个box2-->
        <div class="box1" :class='{box2: isGreen}'></div>
        <!--也允许填充这种数组的形式，会对应的把data里面的值映射过来-->
        <div class='a1' :class='{a2,a3}'>aaa</div>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js"></script>
    <script>
        // v-bind可以绑定标签的属性
        var app = new Vue({
            el: '#app',
            data: {
                title: 'Test for test',
                imgSrc: 'https://cn.vuejs.org/images/logo.png',
                isGreen: true,
                a2: 'a2',
                a3: 'a3'
            }
        })
    </script>
</body>
```

**示例2**

```html
<!DOCTYPE html>
<html lang='en'>
<head>
    <meta charset='UTF-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1.0'>
    <meta http-equiv='X-UA-Compatible' content='ie=edge'>
    <title>Document</title>
    <style>
        .red{
            color: red;
        }
        .thin{
            font-weight: 200;
        }
        .italic{
            font-style: italic;
        }
        .active{
            letter-spacing: 0.5em; /* 中文用letter-spacing定义间距 */
        }
    </style>
</head>
<body>
    <div id='app'>
        <!--以数组的形式传递，而且类名要用单引号抱起来，否则会被认为是变量在data中去寻找。-->
        <h1 :class="['thin', 'red']">this is h1</h1>
        <!--在数组中使用3元表达式，但是这种也不是很常用的用法-->
        <h1 :class="['thin', 'red', flag?'active':'']">this is h1</h1>
        <!--三元表达式也可以改造成一个对象的形式-->
        <h1 :class="['thin', 'red', {'active':flag}]">this is h1</h1>
        <!--直接使用对象的形式，对象的属性可以带引号也可以不带引号。-->
        <h1 :class="{red:true,thin:false,italic:true,active:false}">this is h1</h1>
        <!--直接使用对象的形式其实就等价于直接绑定到data里面，所以这里直接填写data中的一个对象属性也是ok的。-->
        <h1 :class="classObj">this is h1</h1>
    </div>
    <script src='https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js'></script>
    <script>
        var app = new Vue({
            el: '#app',
            data: {
                flag: true,
                classObj: { red: false, thin: false, italic: true, active: false },
            },
            methods: {},
            created: {},
            computed: {},
        })
    </script>
</body>
</html>
```

### v-if & v-show

```html
<body>
    <div id='app'>
        <!--
            v-if和v-show唯一的区别就是，v-if是真正的元素的销毁和重建，底层是通过appendchild和removeChild来实现的。
            而v-show是单纯的display:none这种样式的变化，对应的v-if有较高的切换性能消耗
            而v-show有较高的初始渲染消耗
            如果元素涉及到频繁的切换，最好不要使用v-if
        -->
        <input type="button" value="切换" @click="toggle">
        <h3 v-if="flag">这是用v-if控制的元素</h3>
        <h3 v-show="flag">这是用v-show控制的元素</h3>
    </div>
    <script src='https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js'></script>
    <script>
        var app = new Vue({
            el: '#app',
            data: {
                flag: true,
            },
            methods: {
                toggle(){
                    console.log(this.flag);
                    this.flag = !this.flag;
                }
            },
        })
    </script>
</body>
```

常见的v-if的使用方式

```html
<h1 v-if="role==='lamber'"></h1>
<h1 v-else-if="role==='maxiaoyu'"></h1>
<h1 v-else></h1>
```

### v-on

v-on指定可以实现事件的绑定，类似于之前的onclick，onmouseover这类的操作。

**示例1**

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
    <div id="app">
        <div v-if='show'>
            hahaha
        </div>
        <!--v-on可以做事件的绑定，当然v-on可以简写成一个@符号-->
        <!--对应的方法在Vue对象中的methods中，等号后面的认为是js代码中的一个属性，是一个方法-->
        <input type="button" value="按钮" v-on:click="alerthello">
        <input type="button" value="按钮2" @click="alertbutton">
        <input type="button" value="按钮3" @click="clickHandler">
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el: '#app',
            data: {

            },
            methods: {
                // methods属性中定义了当前vue实例中所有可用的方法
                alerthello(){
                    alert('hello');
                },
                alertbutton(){
                    alert("button");
                }，
                // 这就是对象的一种单体模式，其实就等同于clickHandler: function(){};
                // 在这个模式下，this就是指的当前的实例化对象
                clickHandler(){
                    // 每点一次都是取反，true变成false，false变成true
                    this.show = !this.show
                }
            }
        })
    </script>
</body>
</html>
```

`v-on:事件名称`比如要绑定click事件就是`v-on:click="alertmsg"`，当然还有一个更简便的写法就是`@`符号，可以直接`@click="alertmsg"`也是没有问题的。

**示例2**

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
    <div id="app">
        <!--初始值的count是0，click点击的时候count = count + 1，count发生变化后，后面立即会变-->
        <!--数据驱动视图展示-->
        <button v-on:click="count+=1">当前的count为：{{ count }}</button>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js"></script>
    <script>
        var app  = new Vue({
            el:'#app',
            data : {
                count : 0,
            }
        })
    </script>
</body>
</html>
```



