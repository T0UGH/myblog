---
title: '[javascript][11][js模块]'
date: 2019-06-05 15:09:05
tags:
    - JavaScript
    - ES6
categories:
    - JavaScript
---
## 11 代码模块化

小的、组织良好的代码远比庞大的代码更容易**理解和维护**。优化程序结构和组织方式的一种方式是将代码拆分为小的、耦合相对松散的片段或模块

**模块**是比对象或者函数稍大的、用于组织代码的单元，通过模块可以将程序进行分类

**模块化**的好处
1. 降低理解成本
2. 模块易于维护
3. 提高代码的重用性

### 11.1 在ES6之前的版本中模块化代码

#### 11.1.1 使用对象、闭包和立即执行函数实现模块

ES6之前没有模块，开发者创造性的发挥JS语言现有的特性实现模块化。最流行的方式是使用立即执行函数的闭包实现模块

````js
const MouseCounterModule = function(){
    let numClicks = 0;
    const handleClick = () => {
        alert(++numClicks);
    };
    return {
        countClicks: () => {
            document.addEventListener("click", handleClick);
        }
    };
}();
````
1. 在立即执行函数内部，定义模块内部的实现细节，这些细节只能在模块内部访问:局部变量`numClicks`和局部函数`handleClick`
2. 创建并返回一个对象作为模块的"公共接口"。该接口包括`countClicks`方法
3. 由于暴露了模块接口，模块内部细节仍然可以通过接口创建的闭包保持活跃
4. 最后，保存代表模块接口的变量，通过立即执行函数返回给变量`MouseCounterModule`

这种在JavaScript中通过使用立即执行函数、对象和闭包来创建模块的方式称为**模块模式**

模块模式的问题
1. 难以扩展
2. 模块模式无法很好的处理对于其他模块的依赖

#### 11.1.2 使用AMD模块化JavaScript应用

AMD的设计理念是明确基于浏览器，而CommonJS的设计是面向通用JavaScript环境，而不局限于浏览器

AMD提供名为`define`的函数，它接受如下参数
- 先创建模块的`id`，使用该`id`，可以在系统的其他部分引入该模块
- 当前模块依赖的模块`id`列表
- 初始化模块的工厂函数，该工厂函数接收依赖的模块列表作为参数

AMD定义模块示例
````javaScript
define('MouseCounterModule`, ['jquery'], $ => {
    let numClicks = 0;
    const handleClick = () => {
        alert(++numClicks);
    };
    return {
        countClicks: () => {
            document.addEventListener("click", handleClick);
        }
    };
});
````
#### 11.1.3 使用CommonJS模块化JavaScript应用

CommonJS提供`module`变量，通过`module.exports`对象暴露的对象或函数可以在模块外部访问

````js
const $ = require("jquery");
let numClicks = 0;
const handleClick = () => {
        alert(++numClicks);
};
module.exports = {
    countClicks: () => {
            document.addEventListener("click", handleClick);
    }
};
````

````js
const MouseCounterModule = require("MouseCounterModule.js");
MouseCounterModule.countClicks();
````
### 11.2 ES6模块

ES6模块的主要思想是必须显式地使用标识符导出模块，才能从外部访问模块

ES6引入了两个关键字
- export: 从模块外部指定标识符
- import: 导入模块标识符

#### 11.2.1 导出和导入功能

通过在需要导出的标识符(如变量、函数和类)使用关键字`export`，就可以实现模块导出
````js
const ninja = "Yoshi";
//导出变量
export const message = "Hello";
//导出函数
export function sayHiToNinja(){
    return message + " " + ninja;
}
````

使用关键字`import`将变量和函数导入
````js
import {message, sayHiToNinja} from "Ninja.js"

console.log(message);
sayHiToNinja();
````

使用简化符号`*`可以导入全部标识符
````js
import * as ninjaModule from "Ninja.js"
console.log(ninjaModule.message);
ninjaModule.sayHiToNinja();
````

#### 11.2.2 默认导出

通常我们不需要从模块中导出一组相关的标识符，只需要导出一个标识符来代表整个模块的导出，常见情况为，当模块中包含一个类，只需要导出整个类

````js
export default class Ninja{
    constructor(name){
        this.name = name;
    }
}

export function compareNinjas(ninja1, ninja2){
    return ninja1.name === ninja2.name;
}
````

导入模块默认导出的内容，不需要使用花括号{},可以任意指定名称
````js
import ImportedNinja from "Ninja.js"
import {compareNinjas} from "Ninja.js"

const ninja1 = new ImportedNinja("Yoshi");
const ninja2 = new ImportedNinja("Hattori");

console.log(ninja1 === ninja2);
````

#### 11.2.3 模块语法表

|代码|含义|
|--|--|
|`export const ninja = "Yoshi"`|导出变量|
|`export function compare()`|导出函数|
|`export class Ninja()`|导出类|
|`export default class Ninja()`|导出默认类|
|`export default function Ninja()`|导出默认函数|
|`const ninja = "Yoshi"; function compare(){}; export{ninja, compare};`|导出存在的变量|
|`export {ninja as samurai};`|使用别名导出变量|
|`import Ninja from "Ninja.js"`|导入默认导出|
|`import {ninja} from "Ninja.js"`|导入命名导出|
|`import * as Ninja from "Ninja.js"`|导入模块声明的全部导出内容|
|`import {ninja as iNinja from "Ninja.js"`|通过别名导入模块中声明的全部导出内容|
