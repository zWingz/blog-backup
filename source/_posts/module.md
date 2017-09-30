---
title: 个人对Js模块加载的理解
date: 2017-02-08 18:56:02
tags: [前端,Js]
---
模块加载当前有的规范是
`AMD`、`CMD`、`CommonJs`、`UMD`以及es6的`Module`。

AMD和CMD用在浏览器端，CommonJs用来客户端，而UMD则是柔和AMD和CommonJs，使得前后端可以共用一套模块加载方案。

AMD和CMD原理是通过scrpit标签的异步加载完成

CommonJs利用node中的四个变量来完成 module、exports、require、global。

<!-- more -->
**Amd 是requirejs引出来的一套加载规范**
```javascript
define(id?, dependencies?, factory);
```
id: 模块标识，可以省略。
dependencies: 所依赖的模块，可以省略。
factory: 模块的实现，或者一个JavaScript对象。  

```javascript
//b.js 依赖a.js
define(['a'], function(a) {
    return {
        show: function() {
            ...
        }
    }
});  
```
**Cmd 则是seajs引出来的一套加载规范**

与AMD不同点是
1.AMD对于依赖的模块是提前执行，而CMD是延迟执行。
2.CMD 推崇依赖就近，AMD 推崇依赖前置
```javascript
// CMD
define(function(require, exports, module) {
var a = require('./a')
a.doSomething()
// 此处略去 100 行
var b = require('./b') // 依赖可以就近书写
b.doSomething()
// ... 
})

// AMD 默认推荐的是
define(['./a', './b'], function(a, b) { // 依赖必须一开始就写好
a.doSomething()
// 此处略去 100 行
b.doSomething()
...
}) 
```

**UMD先判断是否支持Node.js的模块（exports）是否存在，存在则使用Node.js模块模式。**
```javascript
(function (root, factory) {
    if (typeof define === 'function' && define.amd) {
        // AMD
        define(['jquery'], factory);
    } else if (typeof exports === 'object') {
        // Commonjs
        module.exports = factory(require('jquery'));
    } else {
        // 赋值给window
        root.returnExports = factory(root.jQuery);
    }
}(this, function ($) {
    //    methods
    function myFunc(){};

    //    exposed public method
    return myFunc;
}));
```


**ES6的Module。定义JS模块加载的一套标准。**

关键字： `import` ，`export`， `default`，`*` ， `as` ;

使用`import`来加载模块,使用`export`来导出模块。

`default`用在export作为默认导出。

`* `作为 import 时导入模块全部输出。

`as` 作为别名。

ES6的模块中始终使用严格模式。

具体使用请参照es6规范。


**ES6 Module和commonJs的区别**

ES6模块加载的机制，与CommonJS模块完全不同。CommonJS模块输出的是一个值的拷贝，而ES6模块输出的是值的引用。

CommonJS中模块一旦输出一个值，模块内部的变化就影响不到这个值。
其实就是将模块的代码跑一遍.获取module.exports中的对象并将其拷贝.拷贝后的对象绑定在module对象中

在node环境下输入module可以查看该对象内容
```javascript
Module {
  id: '<repl>',
  exports: {},
  parent: undefined,
  filename: null,
  loaded: false,
  children: [],
  paths: 
   [ '/home/zwing/gitProject/module-share/dist/repl/node_modules',
     '/home/zwing/gitProject/module-share/dist/node_modules',
     '/home/zwing/gitProject/module-share/node_modules',
     '/home/zwing/gitProject/node_modules',
     '/home/zwing/node_modules',
     '/home/node_modules',
     '/node_modules',
     '/usr/lib/nodejs',
     '/usr/lib/node_modules',
     '/usr/share/javascript',
     '/home/zwing/.node_modules',
     '/home/zwing/.node_libraries',
     '/home/zwing/.nvm/versions/node/v7.0.0/lib/node' ] }

```
在引入一个文件后
```javascript
require（'./a.js'）
```
```javascript
Module {
  id: '<repl>',
  exports: {},
  parent: undefined,
  filename: null,
  loaded: false,
  children: 
   [ Module {
       id: '/home/zwing/gitProject/module-share/dist/a.js',
       exports: {},
       parent: [Circular],
       filename: '/home/zwing/gitProject/module-share/dist/a.js',
       loaded: true,
       children: [Object],
       paths: [Object] } ],
  paths: 
   [ '/home/zwing/gitProject/module-share/dist/repl/node_modules',
     '/home/zwing/gitProject/module-share/dist/node_modules',
     '/home/zwing/gitProject/module-share/node_modules',
     '/home/zwing/gitProject/node_modules',
     '/home/zwing/node_modules',
     '/home/node_modules',
     '/node_modules',
     '/usr/lib/nodejs',
     '/usr/lib/node_modules',
     '/usr/share/javascript',
     '/home/zwing/.node_modules',
     '/home/zwing/.node_libraries',
     '/home/zwing/.nvm/versions/node/v7.0.0/lib/node' ] }

```
即所有的模块都会保存在`module`对象当中,原模块的改变就对已存在与module中的模块不会产生影响


而ES6模块不会缓存运行结果，而是动态地去被加载的模块取值，并且变量总是绑定其所在的模块。
执行模块,并且生成一个动态的只读引用,需要用的时候再去拿,该对象绑定在其所在的模块。
（node暂时还不支持Es6的Module）

还有一点
commonjs中,如果引用一个模块,会把整个模块的exports值都缓存下来.然后再拿相应的东西
module中时,要拿个就拿哪个.其余的不拿.

代码分析:
因此,对于以下代码,在CommonJs和ES6 Module执行是不一样的.
```javascript
// CommonJs

// a.js
var counter = 3;
function incCounter() {
  counter++;
}
module.exports = {
  counter: counter,
  incCounter: incCounter,
};

// main.js
var mod = require('./lib');

console.log(mod.counter);  // 3
mod.incCounter();
console.log(mod.counter); // 3
```
当a模块导出后,其导出的对象被main.js上下文中的module缓存,因此a模块里做任何的更改都不会影响到main

如果要想a中的值能影响到main,只能将counter改写成一个getter函数.
```javascript
var counter = 3;
function incCounter() {
  counter++;
}
module.exports = {
  get counter() {
    return counter
  },
  incCounter: incCounter,
};

```
此时main中获取counter得值会受a模块的影响.


对于Es6 Module(由于Node暂不支持,可以使用webpack打包后进行试验,但也只是webpack自己的实现方式)

```javascript
// a.js
export let counter = 3;
export function incCounter() {
  counter++;
}

// main.js
import { counter, incCounter } from './lib';
console.log(counter); // 3
incCounter();
console.log(counter); // 4
```
此时main中引入的是模块a的一个引用,而不是缓存,直接与a模块挂钩,因此a模块中的更改会对main有所影响.
但是main中引入a模块的变量是只读的,如果重新赋值会报错.


CommonJs与Module相同的一点是,
不同的脚本加载同一个模块时候,得到的是同一个实例对象.
即实际上同一个模块只加载一次.


对于循环加载,commonjs和module也是不一样的
CommonJS模块的重要特性是加载时执行，即脚本代码在require的时候，
就会全部执行。一旦出现某个模块被"循环加载"，就只输出已经执行的部分，还未执行的部分不会输出。
```javascript
// a.js
exports.done = false;
var b = require('./b.js');
console.log('在 a.js 之中，b.done = %j', b.done);
exports.done = true;
console.log('a.js 执行完毕');


//b.js
exports.done = false;
var a = require('./a.js');
console.log('在 b.js 之中，a.done = %j', a.done);
exports.done = true;
console.log('b.js 执行完毕');


//main.js
var a = require('./a.js');
var b = require('./b.js');
console.log('在 main.js 之中, a.done=%j, b.done=%j', a.done, b.done);

```

结果:
```javascript
在 b.js 之中，a.done = false
b.js 执行完毕
在 a.js 之中，b.done = true
a.js 执行完毕
在 main.js 之中, a.done=true, b.done=true
```

对于module
ES6模块是动态引用,这时候加载进来的变量是原模块中的引用,要求开发者自己保证在取值的时候可以真正渠道值
```javascript
// a.js如下
import {bar} from './b.js';
console.log('a.js');
console.log(bar);
export let foo = 'foo';

// b.js
import {foo} from './a.js';
console.log('b.js');
console.log(foo);
export let bar = 'bar';
```

输出:
```javascript
  b.js // a.js已经执行了.所以会继续往下执行b.js
  undefined // 此时a.js没执行完,所以输出undefined
  a.js //执行玩b就执行a
  bar // 此时能正确拿到b中的值
```


如果是commonjs的话,两个值都是undefined

下面是webpack打包后的代码,这段代码主要用来加载已打包好的模块
![webpack-module](/assets/img/webpack-module.png)


``` javascript
function __webpack_require__(moduleId) {

}
```
这个方法主要用来加载模块,moduleId其实就是webpack自己管理的模块索引

```javascript
var installedModules = {}
```
这个对象就是用来管理已经加载完的模块


**以上是我对Js模块加载的理解,有错还请各位纠正!!**