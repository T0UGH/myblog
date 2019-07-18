---
title: '[vue][2][vue组件]'
date: 2019-07-01 23:13:10
tags:
    - JavaScript
    - vue
categories:
    - vue
---
## 2 Vue组件

组件是什么
- 组件是一段独立的，代表了页面的一个部分的代码片段。
- 它拥有自己的数据、JavaScript脚本，以及样式标签。
- 组件可以包含其他的组件，井且它们之间可以相互通信。
- 组件可以是按钮或者图标这样很小的元素，也可以是一个更大的元素

组件的好处
- 将代码从页面中分离到组件中的主要优势是，负责页面每一部分的代码都很靠近该组件中的其余代码。
- 因此当你想要知道哪个元素有添加事件监听器，不必再在一堆JavaScript文件中搜索相应的选择器，因为JavaScript代码就在对应的HTML旁边
- 而且由于组件是独立的，还可以确保组件中的代码不会影响任何其他组件或产生任何副作用。

### 2.1 组件基础

````html
<div id="app">
    <custom-button></custom-button>
</div>
<script>
const CustomButton = {
    template: '<button>自定义按钮</button>'
};
new Vue({
    el: '#app',
    components: {
        CustomButton
    }
});
</script>
````
这样自定义的按钮就会在页面上显示了

### 2.2 数据、方法和计算属性

- 每个组件可以拥有它们自己的数据、方法和计算属性，以及所有在前面章节中出现过的属性就像`Vue`实例一样。
- 定义组件的对象与我们用来定义`Vue`实例的对象相似，它们在很多地方可以互用。


#### 2.2.1 组件和Vue实例之间的一个细微差别

Vue实例中的`data`属性是一个对象，然而组件中的`data`属性是一个函数。这是因为一个组件可以在同一个页面上被多次引用，你大概不希望它们共享一个`data`对象

### 2.3 传递数据


- 当你开始传递数据到它内部时，组件才真正地展示出力量。可以使用`props`属性来传递数据
    ````html
    <div id="app">
        <color-preview color="red"></color-preview>
        <color-preview color="blue"></color-preview>
    </div>
    <script>
    Vue.compoent('color-preview', {
        template: '<div class="color-preview" :style="stype"></div>',
        props: ['color'],
        computed: {
            style(){
                return {backgroundColor: this.color};
            }
        }
    });

    new Vue({
        el: '#app'
    });
    </script>
    ````

- `Props`是通过HTML属性传入组件的,之后在组件内部,`props`属性的值表示可以传入组件的属性的名称

#### 2.3.1 Prop 验证

- `prop` 可以传递一个简单的数组，来表示组件可以接收的属性的名称，也可以传递一个对象，来描述属性的信息，比如:类型、是否必须、默认值以及用于高级验证的自定义验证函数

- 例如:
    ````js
    Vue.component('price-display',{
        props:{
            price: Number,
            unit: String
        }
    })
    ````
    - 如果`price`不是一个数字， 或者`unit`不是一个字符串，`Vue`就会抛出一个警告。

#### 2.3.2 响应式

在父级实例中设定`prop`的值时，可以使用`v-bind`指令将该`prop`与某个值绑定。那么无论何时只要这个值发生变化， 在组件内任何使用该`prop`的地方都会更新。

#### 2.3.3 数据流和`.sync`修饰符

- 通常情况下, 数据通过`prop`从父级组件传递到子组件中，当父级组件中的数据更新时，传递给子组件的`prop`也会更新。但是不可以在子组件中修改`prop`，这就是所谓的单向下行绑定，防止子组件无意中改变父级组件的状态

- 双向数据绑定: 如果使用`.sync`修饰符，可以实现子组件向父组件传递变化
    ````html
    <count-from-number :number.sync="numberToDisplay">
    </count-from-number>
    ````

- 如果仅仅想要更新从`prop`传入的值，而不关心父级组件的值的更新，可以在一开始的`data`函数中通过`this`来引用`prop`的值，将它复制到`data`对象中

### 2.4 使用插槽(slot)将内容传递给组件

除了将数据作为`prop`传入到组件中,`Vue`也允许传入HTML，可以使用`<slot>`标签

````html
<custom-button>单击我</custom-button>
<script>
    Vue.component('custom-button',{
        template: '<button class="custom-button"><slot></slot></button>'
    });
</script>
````

- 具名插槽: 它允许你在同一个组件中拥有多个插槽

#### 2.4.1 作用域插槽

可以将数据传回slot组件，使父组件中的元素可以访问子组件中的数据

1. 创建一个获取用户信息的组件，而数据的显示则留给父级组件来处理
````js
Vue.component('user-data', {
    template: '<div class="user"><slot :user="user"></slot></div>'
    data: () =>{
        user: undifined
    }
})
````
2. 传递给`<slot>`的属性可以用`<slot-scope>`属性中定义的变量来获取
````
<div>
    <user-data slot-scope="user">
        <p v-if="user">用户名:{% raw %}{{ user.name }}{% endraw %}</p>
    </user-data>
</div>
````

### 2.5 自定义事件

### 2.6 混入

### 2.7 vue-loader和`.vue`文件

### 2.8 非`Prop`属性

### 2.9 组件和`v-for`指令

