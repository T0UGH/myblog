---
title: '[vue][1][vue基础]'
date: 2019-06-20 13:12:26
tags:
    - JavaScript
    - vue
categories:
    - vue
---
## 1 Vue.js 基础

参考
1. 《Vue.js快跑构建触手可及的高性能Web应用》
2. [vue官方文档](https://cn.vuejs.org/v2/guide/)

### 1.1 为什么选择Vue.js

#### 1.1.1 简介

- Vue是一套用于构建用户界面的渐进式框架
- Vue被设计为可以自底向上逐层应用
- Vue的核心库只关注视图层
    1. 便于与第三方库或既有项目整合
    2. 当与现代化的工具链以及各种支持类库结合使用时，Vue也完全能够为复杂的单页应用提供驱动
- vue实现了应用逻辑和视图逻辑的完全隔离

#### 1.1.2 相同的功能用jquery和vue分别实现
1. `vue`实现
    ````html
    <ul class=” js-items”>
        <li v-if="!items.length">Sorry, there are no items.</li>
        <li v-for=" item in items" :class="{'is-blue':item.includes('blue')}">
            {% raw %}{{ item }}{% endraw %}
        </li>
    </ul>
    <script>
        new Vue({
            el: '.js-items',
            data: {
                items: []
            },
            created(){
                fetch('https://example.com/items.json')
                .then(res => res.json())
                .then(data => {
                    this.items = data.items;
                });
            }
        });
    </script>
    ````
2. `jquery`实现
    ````html
    <ul class=” js-items ”></Ul>
    <script>
        $(function () {
            $.get ('https://example.com/items.json')
            .then(function (data) {
                var $itemsUl = $('.js-items');
                if (!data.items.length) {
                    var $noitems = $('li');
                    $noitems.text('Sorry, there are no items.');
                    $itemUI.append($noitems);
                }else{
                    data.items.forEach(function(item){
                        var $newItem = $('li');
                        $newItem.text(item);
                        if(item.includes('blue')){
                            $newItem.addClass('is-blue');
                        }
                        $itemUI.append($newItem);
                    })
                }
            });
    ````

### 1.2 安装和设置

- vue-loader:是一个 `webpack` 的加载器，允许你将一个组件的所有 `HTML` 、`JavaScript` 和`CSS`代码编写到同一个文件中

### 1.3 模板(Template)、数据(Data)和指令(Directive)

- `Vue`的核心是将数据显示在页面上，这一功能通过模板实现。为正常的`HTML`添加特殊的属性-被称作指令-借助它来告诉`Vue`我们想要实现的效果以及如何处理提供给它数据。
    ````html
    <div id="app" >
    <p v-if="hours < 12">早上好！</p>
    <p v-if="hours >= 12 && hours < 18">中午好！</p>
    <p v-if="hours >= 18">晚上好！</p>
    <script>
        new Vue({
            el: '#app',
            data: {
                hours: new Date().getHours()
        });
    </script>
    ````

### 1.4 v-if vs v-show

- `v-if`: 假如取值为假，则DOM不会生成

- `v-show`: 假如取值为假，DOM会生成只是此DOM被隐藏

- 使用 `v-if` 会有性能开销。每次插入或者移除元素时都必须要生成元素内部的 `DOM` 树，这在某些时候是非常大的工作量。而 `v-show` 除了在初始创建开销时之外没有额外的开销。如果希望频繁地切换某些内容，那么`v-show`会是最好的选择。

### 1.5 模板中的循环(v-for)

- `v-for`指令通过遍历一个数组或者对象，将指令所绑定的元素循环输出到页面上
    ````html
    <div id ="app">
        <ul>
            <li v-for="dog in dogs">{% raw %}{{ dog }}{% endraw %}</li>
        </ul>
    </div>
    <script>
    new Vue({
        el :'#app',
        data: {
            dogs: ['Rex', 'Rove', 'Hen', 'Alan'］
        }
    });
    </script>
    ````

- `v-for`中键值的参数顺序是(value, key)

### 1.6 属性绑定(v-bind)

- `v-bind`用于将一个值绑定在一个HTML属性上
    ````html
    <div id="app">
        <button v-bind:disabled="buttonDisabled">Test button</button>
    </div>
    <script>
        new Vue({
            el :'#app',
            data: {
                buttonDisabled: true
            }
        });
    </script>
    ````
- `v-bind`可以简写，直接用`:`代替
    ````html
    <button :disabled="buttonDisabled">Test Button</button>
    ````

### 1.7 响应式

Vue监控`data`对象的变化，井在数据变化时更新DOM。

### 1.8 双向数据绑定(v-model)

- `v-model`作用于输入框元素，将输入框的值绑定到`data`对象的对应属性上，
- 因此输入框不但会接收`data`的初始值，而且当输入内容更新时,`data`上的属性值也会更新
- 使用 `v-model` 时一定要记住，如果设置了`value`、`checked`和`selected`属性，这些属性会被忽略。如果想设置输入元素的初始值， 应该在`data`对象中设置

````html
<div id="app">
    <input type="text" v-model="inputText">
    <p>inputText: {% raw %}{{ inputText }}{% endraw %}</p>
</div>
````

### 1.9 动态设置HTML

如果想将HTML渲染到页面上，可以像下面这样使用`v-html`指令：

````html
<div v-html='yourHtml'></div>
````

### 1.9 方法(methods)

- 在Vue的模板里，只要将一个函数存储为`methods`对象的一个属性，就可以在模板中使用它

- 关于方法中的`this`
    - 在方法中，`this`指向该方怯所属的组件
    - 可以使用`this`访问`data`对象的属性和其他方法

````html
<div id="app">
<p>当前状态: {% raw %}{{ statusFromId(status) }}{% endraw %}</p>
</div>
<script>
    new Vue({
        el: '#app',
        data:{
            status: 2
        },
        methods:{
            statusFromId(id) {
                const status= ({
                    0 : '吃包子',
                    1 : '睡觉',
                    2 : '打包子'
                })[id];
                return status || '未知状态' + id;
            }
        },
    });
</script>
````

### 1.10 计算属性(computed)

#### 1.10.1 计算属性
- 计算属性介于`data`和`method`两者之间：可以像访问`data`对象的属性那样访问它，但需要以函数的方式定义它。
- 计算属性会被缓存
    - 如果在模板中多次调用一个方法，方法中的代码在每一次调用时都会执行一遍
    - 但如果计算属性被多次调用，其中的代码只会执行一次，之后的每次调用都会使用被缓存的值
    - 只有当计算属性的依赖、发生变化时，代码才会被再次执行
````html
<div id=“ app”>
    <p>数字的总和是:{% raw %}{{ numberTotal}}{% endraw %}</p>
</div>
<script>
    new Vue ({
        el:'#app',
        data:{
            numbers: [5, 8, 3]
        },
        computed: {
            numberTotal() {
                return numbers.reduce((sum, val) => sum + val) ;
            }
        }
    });
</script>
````
#### 1.10.2 使用data对象、方法还是计算属性
- `data` 对象最适合纯粹的数据：如果想将数据放在某处，然后在模板、方法或者计算属性中使用，那么可以把它放在`data`对象中。后面也许还会更新它。
- 当你希望为模板添加函数功能时，最好使用方法: 给方法传递数据，然后它们会对数据进行处理，最终可能返回不同的数据结果。
- 计算属性达用于执行更加复杂的表达式，这些表达式往往太长或者需要频繁地重复使用，所以不想在模板中直接使用。计算属性往往和其他计算属性或者 `data` 对象一起使用，基本上就像是 `data` 对象的一个扩展和增强版本

### 1.11 侦听器(watch)

- 侦听器可以监听 `data` 对象属性或者计算属性的变化。
- 侦听器使用起来很简单： 只要设置要监听的属性名就可以。
- 侦听器很适合用于处理异步操作
````js
new Vue({
    el: "#app",
    data: {
        count: 0
    },
    watch: {
        count(){
            //变化时会调用此处
        }
    }
})
````
#### 1.11.1 监听data对象中某个对象的属性

有些时候会将一整个对象存储在 `data` 对象中。为了监听这个对象的属性变化，可以在侦听器的名称中使用`.`操作符，就像访问这个对象属性一样：
````js
new Vue({
    data: {
        formData: {
            username: ""
        }
    },
    watch: {
        'formData.username'(){
            //变化时会调用此处
        }
    }
});
````
#### 1.11.2 获取旧值

当监听的属性发生变化时，侦听器会被传入两个参数：所监听属性的当前值和原来的旧值。这一特性可以用来了解到底发生了什么变化
````js
watch: {
    inputValue( val, oldVal) {
        console.log(val, oldVal);
    }
}
````

#### 1.11.3 深度监听

- 当监听一个对象时，可能想监听整个对象的变化，而不仅仅是某个属性。
- 但在默认情况下，如果你正在监听`formData`对象井且修改了`formData.username` ，对应的侦听器井不会触发，它只在`formData`对象被整个替换时触发。
- 监听整个对象被称作深度监听，通过将`deep`选项设置为`true`来开启这一特性


### 1.12 过滤器(fliter)

#### 1.12.1 过滤器

- 过滤器是一种在模板中处理数据的便捷方式
- 它们特别适合对字符串和数字进行简单的显示变化：例如，将字符串变为正确的大小写格式，或者用更容易阅读的格式显示数字。
- 可以用链式调用的方式在一个表达式中使用多个过滤器。`{% raw %}{{productOneCost | round |formatCost}}{% endraw %}`
- 除了在插值中使用，还可以在`v-bind` 中使用过滤器（当绑定数值到属性时）
- 也可以使用`Vue.filter()`来注册一个全局的过滤器， 而不是将过滤器逐一注册到各个组件上

````html
<div id="app">
<p>商品一花费了：{% raw %}{{productOneCost | formatCost }}{% endraw %}</p>
<p>商品二花费了：{% raw %}{{productTwoCost | formatCost }}{% endraw %}</p>
<p>商品三花费了：{% raw %}{{productThreeCost | formatCost }}{% endraw %}</p>
</div>
<script>
new Vue({
    el :'#app',
    data: {
        productOneCost: 998,
        productTwoCost: 2399,
        productThreeCost: 5300
    },
    filters: {
        formatCost(value) {
            return '$' + (value / 100).toFixed(2);
        }
    }
});
</script>
````
#### 1.12.2 使用过滤器有两个注意事项。
1. 过滤器是组件中唯一不能使用`this` 来访问数据或者方法的地方。
2. 只可以在插值和`v-bind`指令中使用过滤器

### 1.13 使用 ref 直接访问元素(ref)

- 有时你会发现需要直接访问一个`DOM`元素
- 也许你正在使用一个不支持`Vue`的第三方库，或者希望做一些`Vue`自身不能完全处理的事情
- 可以使用 `ref` 直接访问元素，而不需要使用`querySelector`或者其他选择`DOM`节点的原生方法。

#### 1.13.1 ref的用法
1. 在`HTML`中绑定`ref`属性
    ````html
    <canvas ref="myCanvas"></canvas>
    ````
2. 然后在`js`中通过`this.$ref`取得DOM元素
    ````js
    let canvas = this.$refs.myCanvas;
    ````

### 1.14 输入和事件(v-on)

- 可以使用 `v-on` 指令将事件侦听器绑定到元素上。这个指令将事件名称作为参数，然后将事件侦听器作为传入值
- `v-on` 指令同样有一个简写方式。可以将`v-on:click` 简写为`＠click` 

````html
<div id="app-5">
    <p>{% raw %}{{ message }}{% endraw %}</p>
    <button v-on:click="reverseMessage">逆转消息</button>
</div>
<script>
var app5 = new Vue({
    el: '#app-5',
    data: {
        message: 'Hello Vue.js!'
    },
    methods: {
        reverseMessage: function() {
            this.message = this.message.split('').reverse().join('')
        }
    }
})
</script>
````

### 1.15 生命周期钩子

![](vue-1-vue基础/0620_0.png)

- 简介: 生命周期钩子是一系列会在组件生命周期(从组件被创建并添加到DOM到组件被销毁的整个过程的各个阶段)被调用的函数

- 八大生命周期钩子
    - `beforeCreate`在实例初始化前被触发。
    - `created`会在实例初始化之后、被添加到DOM之前触发。
    - `beforeMount`会在元素已经准备好被添加到DOM，但还没有添加的时候触发。
    - `mounted`会在元素创建后触发(但并不一定已经添加到了DOM,可以用`nextTick`来保证这一点)。
    - `beforeUpdate`会在由于数据更新将要对`DOM`做一些更改时触发。
    - `updated`会在`DOM`的更改已经完成后触发。
    - `beforeDestroy` 会在组件即将被销毁井且从DOM上移除时触发。
    - `destroyed` 会在组件被销毁后触发

