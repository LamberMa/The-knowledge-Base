# LifeCycle

![Vue生命周期](http://tuku.dcgamer.top/lifecycle.png?lamber_tuku)

**示例说明**

生命周期钩子 = 生命周期函数 = 生命周期事件

创建期间的声明周期函数：
`new Vue()` →  `init((刚初始化一个空的vue实例对象，此时，这个对象身上只有默认的生命周期函数和默认的事件，其他都还没有创建) )`→ 初始化完成以后会执行beforeCreate → init data & methods（此时data和methods都被初始化完成）→ 进入分支判断

```
有没有el属性？
|-- 有 -- 有没有template属性 
               |-- 没有，编译el的outer html作为模板
                  （即我们圈的地，那个div编译为一个模板字符串，把这个模板字符串渲染为内存中的DOM）
               |-- 有，Compile template into render function
|--- 没有
```

上述操作都是进行模板的编译，这个编译过程都是在内存中执行的还没有在页面中进行挂载 → beforeMount → 然后创建$el这个变量，用来替换el【将内存中编译好的模板放到真实的页面中去】；

然后执行mounted函数，运行期间的声明周期函数：beforeUpdate 和 updated，当数据不发生变更的话，这两个周期函数不会执行，因此这两个函数执行的此时为0~正无穷。
在mounted状态下 → 当data数据发生了改变 → beforeUpdate → 虚拟DOM重新渲染和修补 → updated

销毁期间的声明周期函数：
beforeDestroy -- TearDown watchers，child components and event listeners --- destroyed当执行beforeDestroy的时候，Vue实例就已经从运行阶段进入到了销毁节点，当执行beforeDestroy的时候，实例身上所有的data和所有的methods以及filter，directives都处于可用状态。此时还没有真正的执行销毁的状态，当执行到destroyed的时候才是组件真正的被销毁掉了，上述的所有属性就都不可用了。
            

```html
<!DOCTYPE html>
<html lang='en'>
<head>
    <meta charset='UTF-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1.0'>
    <meta http-equiv='X-UA-Compatible' content='ie=edge'>
    <title>Document</title>
</head>
<body>
    <div id='app'>
        <h3 id='h3'>{{msg}}</h3>
        <input type="button" @click="msg='No'" value='修改msg'>
    </div>
    <script src='https://cdn.jsdelivr.net/npm/vue@2.5.21/dist/vue.js'></script>
    <script>
        var app = new Vue({
            el: '#app',
            data: {
                msg: 'ok'
            },
            methods: {
                show(){
                    console.log('执行了show方法')
                }
            },
            beforeCreate() {
                // 第一个生命周期函数，表示实例完全被创建出来之前会执行它，
                // 在beforeCreate生命周期函数执行的时候，data和methods中的数据都没有被初始化
            },
            created() {
                // 第二个生命周期函数，此时data和methods已经被初始化完成，在created中，
                // 可以进行调用吗，这里是最早可以使用data和methods的地方。（Important）
                // console.log(this.msg)
            },
            beforeMount() {
                // 第三个声明周期函数，表示模板字符串已经在内存中编译完成，但是尚未渲染到页面中去。
                // 可以看到，这里返回的是{{msg}}而并不是我们想要的ok，由此可以得出，
                // 在beforemount执行的时候，页面中的元素还没有被真正的替换过来，仅仅是一些模板字符串。
                // console.log(document.getElementById('h3').innerText)
            },
            mounted(){
                // 第四个声明周期函数，表示内存中的模板已经真实的挂载到了页面中，
                // 此时执行下面的输出就可以看到ok了，而不是未被替换的模板表达式了。
                // mounted是实例创建期间的最后一个生命周期函数，
                // 当执行完mounted就表示实例已经完全被创建完成，此时，如果没有其他的操作的话
                // 这个实例就不会再发生改变。（Important）
                // console.log(document.getElementById('h3').innerText)
            },
            // 运行中的两个事件
            beforeUpdate(){
                // 此时表示data发生了改变，但是还没有被渲染的页面
                console.log('界面上元素的内容:' + document.getElementById('h3').innerText)
                console.log('data中的msg:' + this.msg)
            },
            updated(){
                // 此时虚拟DOM已经更新完成，并且页面被重新渲染了，此时data和页面上的数据都是最新的了。
                console.log('updated界面上元素的内容:' + document.getElementById('h3').innerText)
            },
        })
    </script>
</body>
</html>
```

