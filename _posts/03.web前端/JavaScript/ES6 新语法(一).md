---
title: ES6 新语法 (一)
date: "2019-07-09 13:37:00"
categories:
- web前端
- JavaScript
tags:
- 前端
- JavaScript
toc: true
typora-root-url: ..\..\..
---

### ECMAScript简介

#### 介绍

ES6， 全称 ECMAScript 6.0 ，是 JavaScript 的下一个版本标准，2015.06 发版。

ES6 主要是为了解决 ES5 的先天不足，比如 JavaScript 里并没有类的概念，但是目前浏览器的 JavaScript 是 ES5 版本，大多数高版本的浏览器也支持 ES6，不过只实现了 ES6 的部分特性和功能。

#### ECMAScript 的背景

JavaScript 是大家所了解的语言名称，但是这个语言名称是商标（ Oracle 公司注册的商标）。因此，JavaScript 的正式名称是 ECMAScript 。1996年11月，JavaScript 的创造者网景公司将 JS 提交给国际化标准组织 ECMA（European computer manufactures association，欧洲计算机制造联合会），希望这种语言能够成为国际标准，随后 ECMA 发布了规定浏览器脚本语言的标准，即 ECMAScript。这也有利于这门语言的开放和中立。

#### ECMAScript的历史

ES6 是 ECMAScript 标准十余年来变动最大的一个版本，为其添加了许多新的语法特性。

- 1997 年 ECMAScript 1.0 诞生。
- 1998 年 6 月 ECMAScript 2.0 诞生，包含一些小的更改，用于同步独立的 ISO 国际标准。
- 1999 年 12 月 ECMAScript 3.0诞生，它是一个巨大的成功，在业界得到了广泛的支持，它奠定了 JS 的基本语法，被其后版本完全继承。直到今天，我们一开始学习 JS ，其实就是在学 3.0 版的语法。
- 2000 年的 ECMAScript 4.0 是当下 ES6 的前身，但由于这个版本太过激烈，对 ES 3 做了彻底升级，所以暂时被"和谐"了。
- 2009 年 12 月，ECMAScript 5.0 版正式发布。ECMA 专家组预计 ECMAScript 的第五个版本会在 2013 年中期到 2018 年作为主流的开发标准。2011年6月，ES 5.1 版发布，并且成为 ISO 国际标准。
- 2013 年，ES6 草案冻结，不再添加新的功能，新的功能将被放到 ES7 中；2015年6月， ES6 正式通过，成为国际标准。

### webpack 

#### webpack介绍

webpack 是一个现代 JavaScript 应用程序的静态模块打包器 (module bundler) 。当 webpack 处理应用程序时，它会递归地构建一个依赖关系图 (dependency graph) ，其中包含应用程序需要的每个模块，然后将所有这些模块打包成一个或多个 bundle 。

#### web的是个核心概念

- 入口 (entry)
- 输出 (output)
- loader
- 插件 (plugins)

#### 入口(entry)

入口会指示 webpack 应该使用哪个模块，来作为构建其内部依赖图的开始。进入入口起点后，webpack 会找出有哪些模块和库是入口起点（直接和间接）依赖的。在 webpack 中入口有多种方式来定义，如下面例子：

```javascript
// 单个入口（简写）语法:
const config = {
  entry: "./src/main.js"
}

// 对象语法:
const config = {
  app: "./src/main.js",
  vendors: "./src/vendors.js"
}
```

#### 输出(output)

output 属性会告诉 webpack 在哪里输出它创建的 bundles ，以及如何命名这些文件，默认值为 ./dist:

```javascript
const config = {
  entry: "./src/main.js",
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, 'dist')
  }
}
```

#### loader

loader 让 webpack 可以去处理那些非 JavaScript 文件（ webpack 自身只理解 JavaScript ）。loader 可以将所有类型的文件转换为 webpack 能够有效处理的模块，例如，开发的时候使用 ES6 ，通过 loader 将 ES6 的语法转为 ES5 ，如下配置：

```javascript
const config = {
  entry: "./src/main.js",
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, 'dist')
  },
  module: {
    rules: [
      {
          test: /\.js$/,
          exclude: /node_modules/,
          loader: "babel-loader",
          options: [
            presets: ["env"]
          ]
      }
    ]
  }
}
```

####  插件(plugins)

loader 被用于转换某些类型的模块，而插件则可以做更多的事情。包括打包优化、压缩、定义环境变量等等。插件的功能强大，是 webpack 扩展非常重要的利器，可以用来处理各种各样的任务。使用一个插件也非常容易，只需要 require() ，然后添加到 plugins 数组中。

```javascript
// 通过 npm 安装
const HtmlWebpackPlugin = require('html-webpack-plugin');
// 用于访问内置插件 
const webpack = require('webpack'); 
 
const config = {
  module: {
    rules: [
      {
          test: /\.js$/,
          exclude: /node_modules/,
          loader: "babel-loader"
      }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({template: './src/index.html'})
  ]
};
```

#### 利用 webpack 搭建应用

```javascript
const path = require('path');
 
module.exports = {
  mode: "development", // "production" | "development"
  // 选择 development 为开发模式， production 为生产模式
  entry: "./src/main.js",
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, 'dist')
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: "babel-loader",
        options: [
          presets: ["env"]
        ]
      }
    ]
  },
  plugins: [
    ...
  ]
}
```

### ES6 新语法特性

#### let 与 const

**let 的使用方法**

let 生命的是块级的变量，声明的变量只在 let 命令所在的代码块内有效。

```javascript
{
  let a = 0;
  a   // 0
}
a   // 报错 ReferenceError: a is not defined

{
  let a = 0;
  var b = 1;
}
a  // ReferenceError: a is not defined
b  // 1

// for 循环计数器很适合用 let
for (var i = 0; i < 10; i++) {
  setTimeout(function(){
    console.log(i);
  })
}
// 输出十个 10
for (let j = 0; j < 10; j++) {
  setTimeout(function(){
    console.log(j);
  })
}
// 输出 0123456789
```

变量 i 是用 var 声明的，在全局范围内有效，所以全局中只有一个变量 i, 每次循环时，setTimeout 定时器里面的 i 指的是全局变量 i ，而循环里的十个 setTimeout 是在循环结束后才执行，所以此时的 i 都是 10。

变量 j 是用 let 声明的，当前的 i 只在本轮循环中有效，每次循环的 j 其实都是一个新的变量，所以 setTimeout 定时器里面的 j 其实是不同的变量，即最后输出12345。（若每次循环的变量 j 都是重新声明的，如何知道前一个循环的值？这是因为 JavaScript 引擎内部会记住前一个循环的值）。

**let的特点**

- let 声明的变量代码块内有效；
- 不能重复声明；
- 不存在变量提升，在声明之前饮用会报错；

#### const 命令

const 声明一个只读变量，声明之后不允许改变。意味着，一旦声明必须初始化，否则会报错。

**基本用法**

```javascript
const PI = "3.1415926";
PI  // 3.1415926

const MY_AGE;  // SyntaxError: Missing initializer in const declaration   
```

**暂时性死区**

```javascript
var PI = "a";
if(true){
  console.log(PI);  // ReferenceError: PI is not defined
  const PI = "3.1415926";
}
```

ES6 明确规定，代码块内如果存在 let 或者 const，代码块会对这些命令声明的变量从块的开始就形成一个封闭作用域。代码块内，在声明变量 PI 之前使用它会报错。

### 解构赋值

#### 概念

解构赋值是对赋值运算符的扩展。

他是一种针对数组或者对象进行模式匹配，然后对其中的变量进行赋值。

在代码书写上简洁且易读，语义更加清晰明了；也方便了复杂对象中数据字段获取。

#### 解构模型

在解构中，有下面两部分参与：

解构的源，解构赋值表达式的右边部分。解构的目标，解构赋值表达式的左边部分。

#### 数组模型的解构(Array)

**基本**

```javascript
let [a, b, c] = [1, 2, 3]; // a = 1 // b = 2 // c = 3
```

**可嵌套**

```javascript
let [a, [[b], c]] = [1, [[2], 3]]; // a = 1 // b = 2 // c = 3
```

**可忽略**

```javascript
let [a, , b] = [1, 2, 3]; // a = 1 // b = 3
```

**不完全解构**

```javascript
let [a = 1, b] = []; // a = 1, b = undefined
```

**剩余运算符**

```javascript
let [a, ...b] = [1, 2, 3]; //a = 1 //b = [2, 3]
```

**字符串等**

在数组的解构中，解构的目标若为可遍历对象，皆可进行解构赋值。可遍历对象即实现 Iterator 接口的数据。

```javascript
let [a, b, c, d, e] = 'hello'; // a = 'h' // b = 'e' // c = 'l' // d = 'l' // e = 'o'
```

**解构默认值**

```javascript
let [a = 2] = [undefined]; // a = 2

// 当解构模式有匹配结果，且匹配结果是 undefined 时，会触发默认值作为返回结果。
```

```javascript
let [a = 3, b = a] = [];     // a = 3, b = 3 

let [a = 3, b = a] = [1];    // a = 1, b = 1 

let [a = 3, b = a] = [1, 2]; // a = 1, b = 2
// a 与 b 匹配结果为 undefined ，触发默认值：a = 3; b = a =3
// a 正常解构赋值，匹配结果：a = 1，b 匹配结果 undefined ，触发默认值：b = a =1
// a 与 b 正常解构赋值，匹配结果：a = 1，b = 2
```

#### 对象模型的解构（Object）

**基本**

```javascript
let { foo, bar } = { foo: 'aaa', bar: 'bbb' }; 

// foo = 'aaa' 

// bar = 'bbb'  

let { baz : foo } = { baz : 'ddd' }; 

// foo = 'ddd'
```

**可嵌套可忽略**

```javascript
let obj = {p: ['hello', {y: 'world'}] }; 

let {p: [x, { y }] } = obj;

// x = 'hello' 

// y = 'world' 

let obj = {p: ['hello', {y: 'world'}] }; 

let {p: [x, {  }] } = obj; 

// x = 'hello'
```

**不完全解构**

```javascript
let obj = {p: [{y: 'world'}] }; 

let {p: [{ y }, x ] } = obj; 

// x = undefined 

// y = 'world'
```

**剩余运算符**

```javascript
let {a, b, ...rest} = {a: 10, b: 20, c: 30, d: 40}; 

// a = 10 

// b = 20 

// rest = {c: 30, d: 40}
```

**解构默认值**

```javascript
let {a = 10, b = 5} = {a: 3}; 

// a = 3; b = 5; 

let {a: aa = 10, b: bb = 5} = {a: 3}; 

// aa = 3; bb = 5;
```

### ES6 Symbol

#### 概述

ES6 引入了一种新的原始数据类型 Symbol ，表示独一无二的值，最大的用法是用来定义对象的唯一属性名。

ES6 数据类型除了 Number 、 String 、 Boolean 、 Objec t、 null 和 undefined ，还新增了 Symbol 。

#### 基本用法

Symbol 函数栈不能用 new 命令，因为 Symbol 是原始数据类型，不是对象。可以接受一个字符串作为参数，为新创建的 Symbol 提供描述，用来显示在控制台或者作为字符串的时候使用，便于区分。

```javascript
let sy = Symbol("KK");
console.log(sy);   // Symbol(KK)
typeof(sy);        // "symbol"
 
// 相同参数 Symbol() 返回的值不相等
let sy1 = Symbol("kk"); 
sy === sy1;       // false
```

#### 使用场景

**作为属性名**

由于每一个 Symbol 的值都是不相等的，所以 Symbol 作为对象的属性名，可以保证属性不重名。

```javascript
let sy = Symbol("key1");
 
// 写法1
let syObject = {};
syObject[sy] = "kk";
console.log(syObject);    // {Symbol(key1): "kk"}
 
// 写法2
let syObject = {
  [sy]: "kk"
};
console.log(syObject);    // {Symbol(key1): "kk"}
 
// 写法3
let syObject = {};
Object.defineProperty(syObject, sy, {value: "kk"});
console.log(syObject);   // {Symbol(key1): "kk"}
```

Symbol 作为对象属性名时不能用.运算符，要用方括号。因为.运算符后面是字符串，所以取到的是字符串 sy 属性，而不是 Symbol 值 sy 属性。

```javascript
let syObject = {};
syObject[sy] = "kk";
 
syObject[sy];  // "kk"
syObject.sy;   // undefined
```

**注意点**

Symbol 值作为属性名时，该属性是公有属性不是私有属性，可以在类的外部访问。但是不会出现在 for...in 、 for...of 的循环中，也不会被 Object.keys() 、 Object.getOwnPropertyNames() 返回。如果要读取到一个对象的 Symbol 属性，可以通过 Object.getOwnPropertySymbols() 和 Reflect.ownKeys() 取到。

```javascript
let syObject = {};
syObject[sy] = "kk";
console.log(syObject);
 
for (let i in syObject) {
  console.log(i);
}    // 无输出
 
Object.keys(syObject);                     // []
Object.getOwnPropertySymbols(syObject);    // [Symbol(key1)]
Reflect.ownKeys(syObject);                 // [Symbol(key1)]
```

**定义常量**

在 ES5 使用字符串表示常量。例如：

```javascript
const COLOR_RED = "red";
const COLOR_YELLOW = "yellow";
const COLOR_BLUE = "blue";
```

但是用字符串不能保证常量是独特的，这样会引起一些问题：

```javascript
const COLOR_RED = "red";
const COLOR_YELLOW = "yellow";
const COLOR_BLUE = "blue";
const MY_BLUE = "blue"；
 
function getConstantName(color) {
    switch (color) {
        case COLOR_RED :
            return "COLOR_RED";
        case COLOR_YELLOW :
            return "COLOR_YELLOW ";
        case COLOR_BLUE:
            return "COLOR_BLUE";
        case MY_BLUE:
            return "MY_BLUE";         
        default:
            throw new Exception('Can't find this color');
    }
}
```

但是使用 Symbol 定义常量，这样就可以保证这一组常量的值都不相等。用 Symbol 来修改上面的例子。

```javascript
const COLOR_RED = Symbol("red");
const COLOR_YELLOW = Symbol("yellow");
const COLOR_BLUE = Symbol("blue");
 
function getConstantName(color) {
    switch (color) {
        case COLOR_RED :
            return "COLOR_RED";
        case COLOR_YELLOW :
            return "COLOR_YELLOW ";
        case COLOR_BLUE:
            return "COLOR_BLUE";
        default:
            throw new Exception('Can't find this color');
    }
}
```

Symbol 的值是唯一的，所以不会出现相同值得常量，即可以保证 switch 按照代码预想的方式执行。

**Symbol.for() **

Symbol.for() 类似单例模式，首先会在全局搜索被登记的 Symbol 中是否有该字符串参数作为名称的 Symbol 值，如果有即返回该 Symbol 值，若没有则新建并返回一个以该字符串参数为名称的 Symbol 值，并登记在全局环境中供搜索。

```javascript
let yellow = Symbol("Yellow");
let yellow1 = Symbol.for("Yellow");
yellow === yellow1;      // false
 
let yellow2 = Symbol.for("Yellow");
yellow1 === yellow2;     // true
```

**Symbol.keyFor()**

```javascript
let yellow1 = Symbol.for("Yellow");
Symbol.keyFor(yellow1);    // "Yellow"
```

### ES6 Map 与 Set

#### Map 对象

Map 对象保存键值对。任何值(对象或者原始值) 都可以作为一个键或一个值。

#### Maps 与Objects 的区别

- 一个 Object 的键只能是字符串或者 Symbols，但一个 Map 的键可以是任意值。
- Map 中的键值是有序的（FIFO 原则），而添加到对象中的键则不是。
- Map 的键值对个数可以从 size 属性获取，而 Object 的键值对个数只能手动计算。
- Object 都有自己的原型，原型链上的键名有可能和你自己在对象上的设置的键名产生冲突。

#### Map 中的key

**key 是字符串**

```javascript
var myMap = new Map(); 

var keyString = "a string";    

myMap.set(keyString, "和键'a string'关联的值");   

myMap.get(keyString);    // "和键'a string'关联的值" 

myMap.get("a string");   // "和键'a string'关联的值"  ，因为 keyString === 'a string'
```

**key 是对象**

```javascript
var myMap = new Map(); 

var keyObj = {}    

myMap.set(keyObj, "和键 keyObj 关联的值"); ﻿ 

myMap.get(keyObj); // "和键 keyObj 关联的值" 

myMap.get({}); // undefined, 因为 keyObj !== {}
```

**key 是函数**

```javascript
var myMap = new Map(); 

var keyFunc = function () {}, // 函数   

myMap.set(keyFunc, "和键 keyFunc 关联的值");   

myMap.get(keyFunc); // "和键 keyFunc 关联的值" 

myMap.get(function() {}) // undefined, 因为 keyFunc !== function () {}
```

**key 是 NaN**

```javascript
var myMap = new Map(); 

myMap.set(NaN, "not a number");   

myMap.get(NaN); // "not a number"   

var otherNaN = Number("foo"); 

myMap.get(otherNaN); // "not a number"
// 虽然 NaN 和任何值甚至和自己都不相等(NaN !== NaN 返回true)，NaN作为Map的键来说是没有区别的。
```

#### Map 的迭代

**for...of**

```javascript
var myMap = new Map();
myMap.set(0, "zero");
myMap.set(1, "one");
 
// 将会显示两个 log。 一个是 "0 = zero" 另一个是 "1 = one"
for (var [key, value] of myMap) {
  console.log(key + " = " + value);
}
for (var [key, value] of myMap.entries()) {
  console.log(key + " = " + value);
}
/* 这个 entries 方法返回一个新的 Iterator 对象，它按插入顺序包含了 Map 对象中每个元素的 [key, value] 数组。 */
 
// 将会显示两个log。 一个是 "0" 另一个是 "1"
for (var key of myMap.keys()) {
  console.log(key);
}
/* 这个 keys 方法返回一个新的 Iterator 对象， 它按插入顺序包含了 Map 对象中每个元素的键。 */
 
// 将会显示两个log。 一个是 "zero" 另一个是 "one"
for (var value of myMap.values()) {
  console.log(value);
}
/* 这个 values 方法返回一个新的 Iterator 对象，它按插入顺序包含了 Map 对象中每个元素的值。 */
```

**forEach**

```javascript
var myMap = new Map();
myMap.set(0, "zero");
myMap.set(1, "one");
 
// 将会显示两个 logs。 一个是 "0 = zero" 另一个是 "1 = one"
myMap.forEach(function(value, key) {
  console.log(key + " = " + value);
}, myMap)
```

#### Map对象的操作

**Map 与 Array的转换**

```javascript
var kvArray = [["key1", "value1"], ["key2", "value2"]];
 
// Map 构造函数可以将一个 二维 键值对数组转换成一个 Map 对象
var myMap = new Map(kvArray);
 
// 使用 Array.from 函数可以将一个 Map 对象转换成一个二维键值对数组
var outArray = Array.from(myMap);
```

**Map 的克隆**

```javascript
var myMap1 = new Map([["key1", "value1"], ["key2", "value2"]]);
var myMap2 = new Map(myMap1);
 
console.log(original === clone); 
// 打印 false。 Map 对象构造函数生成实例，迭代出新的对象。
```

**Map 的合并**

```javascript
var first = new Map([[1, 'one'], [2, 'two'], [3, 'three'],]);
var second = new Map([[1, 'uno'], [2, 'dos']]);
 
// 合并两个 Map 对象时，如果有重复的键值，则后面的会覆盖前面的，对应值即 uno，dos， three
var merged = new Map([...first, ...second]);
```

#### Set 对象

Set 对象允许你存储任何类型的唯一值，无论是原始值或者是对象引用。

**Set 中的特殊值**

Set 对象存储的值总是唯一的，所以需要判断两个值是否恒等。有几个特殊值需要特殊对待：

- +0 与 -0 在存储判断唯一性的时候是恒等的，所以不重复；
- undefined 与 undefined 是恒等的，所以不重复；
- NaN 与 NaN 是不恒等的，但是在 Set 中只能存一个，不重复。

**基本使用**

```javascript
let mySet = new Set();
 
mySet.add(1); // Set(1) {1}
mySet.add(5); // Set(2) {1, 5}
mySet.add(5); // Set(2) {1, 5} 这里体现了值的唯一性
mySet.add("some text"); 
// Set(3) {1, 5, "some text"} 这里体现了类型的多样性
var o = {a: 1, b: 2}; 
mySet.add(o);
mySet.add({a: 1, b: 2}); 
// Set(5) {1, 5, "some text", {…}, {…}} 
// 这里体现了对象之间引用不同不恒等，即使值相同，Set 也能存储
```

**Set 类型转换**

```javascript
// Array 转 Set
var mySet = new Set(["value1", "value2", "value3"]);
// 用...操作符，将 Set 转 Array
var myArray = [...mySet];
String
// String 转 Set
var mySet = new Set('hello');  // Set(4) {"h", "e", "l", "o"}
// 注：Set 中 toString 方法是不能将 Set 转换成 String
```

#### Set 对象的作用

**数组去重**

```javascript
var mySet = new Set([1, 2, 3, 4, 4]); 

[...mySet]; // [1, 2, 3, 4]
```

**并集**

```javascript
var a = new Set([1, 2, 3]); 

var b = new Set([4, 3, 2]); 

var union = new Set([...a, ...b]); // {1, 2, 3, 4}
```

**交集**

```javascript
var a = new Set([1, 2, 3]); 

var b = new Set([4, 3, 2]); 

var intersect = new Set([...a].filter(x => b.has(x))); // {2, 3}
```

**差集**

```javascript
var a = new Set([1, 2, 3]); 

var b = new Set([4, 3, 2]); 

var difference = new Set([...a].filter(x => !b.has(x))); // {1}
```

### ES6 Reflect 与 Proxy

#### 概述

Proxy 与 Reflect 是 ES6 为了操作对象引入的 API 。

Proxy 可以对目标对象的读取、函数调用等操作进行拦截，然后进行操作处理。它不直接操作对象，而是像代理模式，通过对象的代理对象进行操作，在进行这些操作时，可以添加一些需要的额外操作。

Reflect 可以用于获取目标对象的行为，它与 Object 类似，但是更易读，为操作对象提供了一种更优雅的方式。它的方法与 Proxy 是对应的。

#### Proxy基本用法

一个 Proxy 对象由两个部分组成： target 、 handler 。

在通过 Proxy 构造函数生成实例对象时，需要提供这两个参数。 target 即目标对象， handler 是一个对象，声明了代理 target 的指定行为。

```javascript
let target = {
    name: 'Tom',
    age: 24
}
let handler = {
    get: function(target, key) {
        console.log('getting '+key);
        return target[key]; // 不是target.key
    },
    set: function(target, key, value) {
        console.log('setting '+key);
        target[key] = value;
    }
}
let proxy = new Proxy(target, handler)
proxy.name     // 实际执行 handler.get
proxy.age = 25 // 实际执行 handler.set
// getting name
// setting age
// 25
 
// target 可以为空对象
let targetEpt = {}
let proxyEpt = new Proxy(targetEpt, handler)
// 调用 get 方法，此时目标对象为空，没有 name 属性
proxyEpt.name // getting name
// 调用 set 方法，向目标对象中添加了 name 属性
proxyEpt.name = 'Tom'
// setting name
// "Tom"
// 再次调用 get ，此时已经存在 name 属性
proxyEpt.name
// getting name
// "Tom"
 
// 通过构造函数新建实例时其实是对目标对象进行了浅拷贝，因此目标对象与代理对象会互相影响
targetEpt)
// {name: "Tom"}
 
// handler 对象也可以为空，相当于不设置拦截操作，直接访问目标对象
let targetEmpty = {}
let proxyEmpty = new Proxy(targetEmpty,{})
proxyEmpty.name = "Tom"
targetEmpty) // {name: "Tom"}
```

#### Proxy实例方法

**get 方法**

```javascript
get(target, propKey, receiver)
```

用于 target 对象上 propKey 的读取操作。

```javascript
let exam ={
    name: "Tom",
    age: 24
}
let proxy = new Proxy(exam, {
  get(target, propKey, receiver) {
    console.log('Getting ' + propKey);
    return target[propKey];
  }
})
proxy.name 
// Getting name
// "Tom"
```

get() 方法可以继承。

```javascript
let proxy = new Proxy({}, {
  get(target, propKey, receiver) {
      // 实现私有属性读取保护
      if(propKey[0] === '_'){
          throw new Erro(`Invalid attempt to get private     "${propKey}"`);
      }
      console.log('Getting ' + propKey);
      return target[propKey];
  }
});
 
let obj = Object.create(proxy);
obj.name
// Getting name
```

**set 方法**

```javascript
set(target, propKey, value, receiver)
```

用于拦截 target 对象上的 propKey 的赋值操作。如果目标对象自身的某个属性，不可写且不可配置，那么set方法将不起作用。

```javascript
let validator = {
    set: function(obj, prop, value) {
        if (prop === 'age') {
            if (!Number.isInteger(value)) {
                throw new TypeError('The age is not an integer');
            }
            if (value > 200) {
                throw new RangeError('The age seems invalid');
            }
        }
        // 对于满足条件的 age 属性以及其他属性，直接保存
        obj[prop] = value;
    }
};
let proxy= new Proxy({}, validator)
proxy.age = 100;
proxy.age           // 100
proxy.age = 'oppps' // 报错
proxy.age = 300     // 报错
```

第四个参数 receiver 表示原始操作行为所在对象，一般是 Proxy 实例本身。

```javascript
const handler = {
    set: function(obj, prop, value, receiver) {
        obj[prop] = receiver;
    }
};
const proxy = new Proxy({}, handler);
proxy.name= 'Tom';
proxy.name=== proxy // true
 
const exam = {}
Object.setPrototypeOf(exam, proxy)
exam.name = "Tom"
exam.name === exam // true
```

注意，严格模式下，set代理如果没有返回true，就会报错。

**apply 方法**

```javascript
apply(target, ctx, args)
```

用于拦截函数的调用、call 和 reply 操作。target 表示目标对象，ctx 表示目标对象上下文，args 表示目标对象的参数数组。

```javascript
function sub(a, b){
    return a - b;
}
let handler = {
    apply: function(target, ctx, args){
        console.log('handle apply');
        return Reflect.apply(...arguments);
    }
}
let proxy = new Proxy(sub, handler)
proxy(2, 1) 
// handle apply
// 1
```

**has 方法**

```javascript
has(target, propKey)
```

用于拦截 HasProperty 操作，即在判断 target 对象是否存在 propKey 属性时，会被这个方法拦截。此方法不判断一个属性是对象自身的属性，还是继承的属性。

```javascript
let  handler = {
    has: function(target, propKey){
        console.log("handle has");
        return propKey in target;
    }
}
let exam = {name: "Tom"}
let proxy = new Proxy(exam, handler)
'name' in proxy
// handle has
// true
```

注意：此方法不拦截 for ... in 循环。

**construct 方法**

```javascript
construct(target, args)
```

用于拦截 new 命令。返回值必须为对象。

```javascript
let handler = {
    construct: function(target, args, newTarget){
        console.log("handle construct");
        return Reflect.construct(target, args, newTarget);
    } }
class exam = {
    constructor(name){
        this.name = name;
    }
}
let proxy = new Proxy(exam,handler)
new proxy("Tom")
// handle construct
// exam {name: "Tom"}
```

**deleteProperty 方法**

```javascript
deleteProperty(target, propKey)
```

用于拦截 delete 操作，如果这个方法抛出错误或者返回 false ，propKey 属性就无法被 delete 命令删除。

**defineProperty 方法**

```javascript
defineProperty(target, propKey, propDesc)
```

用于拦截 Object.definePro若目标对象不可扩展，增加目标对象上不存在的属性会报错；

若属性不可写或不可配置，则不能改变这些属性。

```javascript
let handler = {
    defineProperty: function(target, propKey, propDesc){
        console.log("handle defineProperty");
        return true;
    }
}plet target = {}
let proxy = new Proxy(target, handler)
proxy.name = "Tom"
// handle defineProperty
target
// {name: "Tom"}
 
// defineProperty 返回值为false，添加属性操作无效
let handler1 = {
    defineProperty: function(target, propKey, propDesc){
        console.log("handle defineProperty");
        return false;
    }
}
let target1 = {}
let proxy1 = new Proxy(target1, handler1)
proxy1.name = "Jerry"
target1
// {}
```

**getOwnPropertyDescriptor方法**

```javascript
getOwnPropertyDescriptor(target, propKey)
```

用于拦截 Object.getOwnPropertyD() 返回值为属性描述对象或者 undefined 。

```javascript
let handler = {
    getOwnPropertyDescriptor: function(target, propKey){
        return Object.getOwnPropertyDescriptor(target, propKey);
    }
}ilet target = {name: "Tom"}
let proxy = new Proxy(target, handler)
Object.getOwnPropertyDescriptor(proxy, 'name')
// {value: "Tom", writable: true, enumerable: true, configurable: 
// true}
```

**getPrototypeOf 方法**

```javascript
getPrototypeOf(target)
```

主要用于拦截获取对象原型的操作。包括以下操作：


- Object.prototype._proto_
- Object.prototype.isPrototypeOf()
- Object.getPrototypeOf()
- Reflect.getPrototypeOf()
- instanceof

```javascript
let exam = {}
let proxy = new Proxy({},{
    getPrototypeOf: function(target){
        return exam;
    }
})
Object.getPrototypeOf(proxy) // {}
```

注意，返回值必须是对象或者 null ，否则报错。另外，如果目标对象不可扩展（non-extensible），getPrototypeOf 方法必须返回目标对象的原型对象。

```javascript
let proxy = new Proxy({},{
    getPrototypeOf: function(target){
        return true;
    }
})
Object.getPrototypeOf(proxy)
// TypeError: 'getPrototypeOf' on proxy: trap returned neither object // nor null
```

**isExtensible 方法**

```javascript
isExtensible(target)
```

用于拦截 Object.isExtensible 操作。

该方法只能返回布尔值，否则返回值会被自动转为布尔值。

```javascript
let proxy = new Proxy({},{
    isExtensible:function(target){
        return true;
    }
})
Object.isExtensible(proxy) // true
```

注意：它的返回值必须与目标对象的isExtensible属性保持一致，否则会抛出错误。

```javascript
let proxy = new Proxy({},{
    isExtensible:function(target){
        return false;
    }
})
Object.isExtensible(proxy)
// TypeError: 'isExtensible' on proxy: trap result does not reflect 
// extensibility of proxy target (which is 'true')
```

**ownKeys 方法**

```javascript
ownKeys(target)
```

用于拦截对象自身属性的读取操作。主要包括以下操作：

- Object.getOwnPropertyNames()
- Object.getOwnPropertySymbols()
- Object.keys()
- or...in

方法返回的数组成员，只能是字符串或 Symbol 值，否则会报错。

若目标对象中含有不可配置的属性，则必须将这些属性在结果中返回，否则就会报错。

若目标对象不可扩展，则必须全部返回且只能返回目标对象包含的所有属性，不能包含不存在的属性，否则也会报错。

```javascript
let proxy = new Proxy( {
  name: "Tom",
  age: 24
}, {
    ownKeys(target) {
        return ['name'];
    }
});
Object.keys(proxy)
// [ 'name' ]f返回结果中，三类属性会被过滤：
//          - 目标对象上没有的属性
//          - 属性名为 Symbol 值的属性
//          - 不可遍历的属性
 
let target = {
  name: "Tom",
  [Symbol.for('age')]: 24,
};
// 添加不可遍历属性 'gender'
Object.defineProperty(target, 'gender', {
  enumerable: false,
  configurable: true,
  writable: true,
  value: 'male'
});
let handler = {
    ownKeys(target) {
        return ['name', 'parent', Symbol.for('age'), 'gender'];
    }
};
let proxy = new Proxy(target, handler);
Object.keys(proxy)
// ['name']
```

**preventExtensions 方法**

```javascript
preventExtensions(target)
```

拦截 Object.preventExtensions 操作。

该方法必须返回一个布尔值，否则会自动转为布尔值。

```javascript
// 只有目标对象不可扩展时（即 Object.isExtensible(proxy) 为 false ），
// proxy.preventExtensions 才能返回 true ，否则会报错
var proxy = new Proxy({}, {
  preventExtensions: function(target) {
    return true;
  }
});
// 由于 proxy.preventExtensions 返回 true，此处也会返回 true，因此会报错
Object.preventExtensions(proxy) 被// TypeError: 'preventExtensions' on proxy: trap returned truish but // the proxy target is extensible
 
// 解决方案
 var proxy = new Proxy({}, {
  preventExtensions: function(target) {
    // 返回前先调用 Object.preventExtensions
    Object.preventExtensions(target);
    return true;
  }
});
Object.preventExtensions(proxy)
// Proxy {}
```

**setPrototypeOf 方法**

```javascript
setPrototypeOf
```

主要用来拦截 Object.setPrototypeOf 方法。

返回值必须为布尔值，否则会被自动转为布尔值。

若目标对象不可扩展，setPrototypeOf 方法不得改变目标对象的原型。

```javascript
let proto = {}
let proxy = new Proxy(function () {}, {
    setPrototypeOf: function(target, proto) {
        console.log("setPrototypeOf");
        return true;
    }
}
);
Object.setPrototypeOf(proxy, proto);
// setPrototypeOf
```

**revocable 方法**

用于返回一个可取消的 Proxy 实例。

```javascript
let {proxy, revoke} = Proxy.revocable({}, {});
proxy.name = "Tom";
revoke();
proxy.name 
// TypeError: Cannot perform 'get' on a proxy that has been revoked
```

#### Reflect介绍 

ES6 中将 Object 的一些明显属于语言内部的方法移植到了 Reflect 对象上（当前某些方法会同时存在于 Object 和 Reflect 对象上），未来的新方法会只部署在 Reflect 对象上。

Reflect 对象对某些方法的返回结果进行了修改，使其更合理。

Reflect 对象使用函数的方式实现了 Object 的命令式操作。

#### Reflect静态方法

**Reflect.get**

```javascript
Reflect.get(target, name, receiver)
```

查找并返回 target 对象的 name 属性。

```javascript
let exam = {
    name: "Tom",
    age: 24,
    get info(){
        return this.name + this.age;
    }
}
Reflect.get(exam, 'name'); // "Tom"
 
// 当 target 对象中存在 name 属性的 getter 方法， getter 方法的 this 会绑定 // receiver
let receiver = {
    name: "Jerry",
    age: 20
}
Reflect.get(exam, 'info', receiver); // Jerry20
 
// 当 name 为不存在于 target 对象的属性时，返回 undefined
Reflect.get(exam, 'birth'); // undefined
 
// 当 target 不是对象时，会报错
Reflect.get(1, 'name'); // TypeError
```

**Reflect.set**

```javascript
Reflect.set(target, name, value, receiver)
```

将 target 的 name 属性设置为 value。返回值为 boolean ，true 表示修改成功，false 表示失败。当 target 为不存在的对象时，会报错。

```javascript
let exam = {
    name: "Tom",
    age: 24,
    set info(value){
        return this.age = value;
    }
}
exam.age; // 24
Reflect.set(exam, 'age', 25); // true
exam.age; // 25
 
// value 为空时会将 name 属性清除
Reflect.set(exam, 'age', ); // true
exam.age; // undefined
 
// 当 target 对象中存在 name 属性 setter 方法时，setter 方法中的 this 会绑定 // receiver , 所以修改的实际上是 receiver 的属性,
let receiver = {
    age: 18
}
Reflect.set(exam, 'info', 1, receiver); // true
receiver.age; // 1
 
let receiver1 = {
    name: 'oppps'
}
Reflect.set(exam, 'info', 1, receiver1);
receiver1.age; // 1
```

**Reflect.has**

```javascript
Reflect.has(obj, name)
```

是 name in obj 指令的函数化，用于查找 name 属性在 obj 对象中是否存在。返回值为 boolean。如果 obj 不是对象则会报错 TypeError。

```javascript
let exam = {
    name: "Tom",
    age: 24
}
Reflect.has(exam, 'name'); // true
```

**Reflect.deleteProperty**

```
Reflect.deleteProperty(obj, property)
```

是 delete obj[property] 的函数化，用于删除 obj 对象的 property 属性，返回值为 boolean。如果 obj 不是对象则会报错 TypeError。

```javascript
let exam = {
    name: "Tom",
    age: 24
}
Reflect.deleteProperty(exam , 'name'); // true
exam // {age: 24} 
// property 不存在时，也会返回 true
Reflect.deleteProperty(exam , 'name'); // true
```

**Reflect.construct**

```
Reflect.construct(obj, args)
```

等同于 new target(...args)。

```javascript
function exam(name){
    this.name = name;
}
Reflect.construct(exam, ['Tom']); // exam {name: "Tom"}
```

**Reflect.getPrototypeOf**

```javascript
Reflect.getPrototypeOf(obj)
```

用于读取 obj 的 \_proto_ 属性。在 obj 不是对象时不会像 Object 一样把 obj 转为对象，而是会报错。

```javascript
class Exam{}
let obj = new Exam()
Reflect.getPrototypeOf(obj) === Exam.prototype // true
```

**Reflect.setPrototypeOf**

```javascript
Reflect.setPrototypeOf(obj, newProto)
```

用于设置目标对象的 prototype。

```javascript
let obj ={}
Reflect.setPrototypeOf(obj, Array.prototype); // true
```

**Reflect.apply**

```javascript
Reflect.apply(func, thisArg, args)
```

等同于 Function.prototype.apply.call(func, thisArg, args) 。func 表示目标函数；thisArg 表示目标函数绑定的 this 对象；args 表示目标函数调用时传入的参数列表，可以是数组或类似数组的对象。若目标函数无法调用，会抛出 TypeError 。

```javascript
Reflect.apply(Math.max, Math, [1, 3, 5, 3, 1]); // 5
```

**Reflect.defineProperty**

```javascript
Reflect.defineProperty(target, propertyKey, attributes)
```

用于为目标对象定义属性。如果 target 不是对象，会抛出错误。

```javascript
let myDate= {}
Reflect.defineProperty(MyDate, 'now', {
  value: () => Date.now()
}); // true
```

**Reflect.getOwnPropertyDescriptor**

```javascript
Reflect.getOwnPropertyDescriptor(target, propertyKey)
```

用于得到 target 对象的 propertyKey 属性的描述对象。在 target 不是对象时，会抛出错误表示参数非法，不会将非对象转换为对象。

```javascript
var exam = {}
Reflect.defineProperty(exam, 'name', {
  value: true,
  enumerable: false,
})
Reflect.getOwnPropertyDescriptor(exam, 'name')
// { configurable: false, enumerable: false, value: true, writable:
// false}
 
 
// propertyKey 属性在 target 对象中不存在时，返回 undefined
Reflect.getOwnPropertyDescriptor(exam, 'age') // undefined
```

**Reflect.isExtensible**

```javascript
Reflect.isExtensible(target)
```

用于判断 target 对象是否可扩展。返回值为 boolean 。如果 target 参数不是对象，会抛出错误。

```javascript
let exam = {}
Reflect.isExtensible(exam) // true
```

**Reflect.preventExtensions**

```javascript
Reflect.preventExtensions(target)
```

用于让 target 对象变为不可扩展。如果 target 参数不是对象，会抛出错误。

```javascript
let exam = {}
Reflect.preventExtensions(exam) // true
```

**Reflect.ownKeys**

```javascript
Reflect.ownKeys(target)
```

用于返回 target 对象的所有属性，等同于 Object.getOwnPropertyNames 与Object.getOwnPropertySymbols 之和。

```javascript
var exam = {
  name: 1,
  [Symbol.for('age')]: 4
}
Reflect.ownKeys(exam) // ["name", Symbol(age)]
```

#### Proxy 和 Reflect组合使用

Reflect 对象的方法与 Proxy 对象的方法是一一对应的。所以 Proxy 对象的方法可以通过调用 Reflect 对象的方法获取默认行为，然后进行额外操作。

```javascript
let exam = {
    name: "Tom",
    age: 24
}
let handler = {
    get: function(target, key){
        console.log("getting "+key);
        return Reflect.get(target, key);
    },
    set: function(target, key, value){
        console.log("setting "+key+" to "+value)
        Reflect.set(target, key, value);
    }
}
let proxy = new Proxy(exam, handler)
proxy.name = "Jerry"
proxy.name
// setting name to Jerry
// getting name
// "Jerry"
```

**使用场景拓展**

```javascript
// 定义 Set 集合
const queuedObservers = new Set();
// 把观察者函数都放入 Set 集合中
const observe = fn => queuedObservers.add(fn);
// observable 返回原始对象的代理，拦截赋值操作
const observable = obj => new Proxy(obj, {set});
function set(target, key, value, receiver) {
  // 获取对象的赋值操作
  const result = Reflect.set(target, key, value, receiver);
  // 执行所有观察者
  queuedObservers.forEach(observer => observer());
  // 执行赋值操作
  return result;
}
```

