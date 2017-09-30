---
title: webpack打包文件的hash问题浅谈
date: 2017-04-28 17:49:10
tags: [前端,问题及思考]
---

**背景**

webpack大家一般都包括以下配置

```javascript
    plugins: [
        new webpack.optimize.CommonsChunkPlugin({
            name: 'vendor',
            // chunks: 'vendor'
            minChunks: function(module) {
                return (
                    module.resource && // 当前引用的资源
                    // /\.js$/.test(module.resource) && // 只找js的
                    module.resource.indexOf(
                        path.join(__dirname, './node_modules')
                    ) === 0 // 同存在于node_modules中的
                )
            }
        }),
        new webpack.optimize.CommonsChunkPlugin({
            name: 'manifest'
        })
    ]

```
<!-- more -->

简单说就是把node_modules引用的全都打包进vendor,然后再提取一层manifest出来以保证业务代码改变不导致vendors的hash变化.
但是有没有真正观察过修改业务代码的时候vendors到底有没有改变呢?
刚升级webpack2.0的时候我没有去关注这个问题.
因为升级完我就试着去验证这个问题.
改动某个业务代码.然后重新build一次.发现vendors的hash并没有改变.然后以为一切很正常.
直到有一次我改动业务代码的时候.增加了一个组件.也就是 import 进一个东西.
打包后发现.vendors的hash竟然改变了.百思不得其解.再试了几次发现的确会改变hash.
但这不是必然的.有些组件增删import必然导致vendors的hash变化.有些组件却不会

```javascript
// app.js
import Vue from 'vue';

import test from './src/test.js'

// test.js
console.log('t123123')

export default a = 1;
```

这时候如果单纯的改console.log中的文字,并不会引起vendors的hash改变
但如果我把import删掉.这时候便会引起vendors的hash改变.
在理想状态的话,是不应该影响到vendors的.
但如果我们在test.js import 一个b.js

```javascript
//test.js
console.log('t123123')
import b from './b.js';
export default a = 1;

// b.js
console.log('bbbbbbbbbbbbbb.js')
export default b = 1;
```
这时候却不会影响vendors的hash值.


为什么? 
是玄学吗? 
不存在的!
我个人觉得,如果增删import的组件会导致webpack的tree-shaking不一致的话则引起hash值的变化.
具体原因尚未深入研究 !

**解决方法**
解决方法就是用WebpackMd5Hash插件(使用md5来作为文件的hash)
```javascript
plugins: [
    ...,
    new new WebpackMd5Hash()
]
```
这时候不管如何改业务代码.都不会引起vendors的变化.除非从nodeModules引用了一个新的库.

但是问题就这么解决吗?
并没有!
细心的会发现.除了vendors的hash不变,连manifest的hash也不变!!!!
manifest的作用是什么 ? 
就是将webpack基础代码提取以及对文件的hash进行管理.按道理来说,只要有改动代码.manifest里面的内容必然会改变.
因为最终打包出来的文件必然会有一个文件的hash改变的.


但manifest现在内容改变了hash却没有改变!!!.
(这次真的是玄学了!)
引起的问题就是浏览器拿到的是缓存的manifest.导致代码无法更新.
问题很严重吗?
也不是啦!
因为我们的代码都最终被htmlWebpackPlugin以script标签的形式插入到html里面.这个会根据最终生成的文件名来插入.
所以manifest内容对其没影响.也就是说.代码改变了.html插入的script也会改变.浏览器也会重新下载新的代码文件.

那是不是就不用管了呢?
答案是否定!
如果我们打包出来的文件是被webpack通过代码分割的形式打包出来.也就是通过

```javascript
require.ensure([], () => {})
```

的形式分割出来的chunks的话
它并不会被htmlWebpackPlugin主动插入到html中.这时候他的加载就要依赖manifest中管理的hash了

所以说如果代码中不包括代码分割的话.可以不考虑manifest的hash.毕竟只用到他的基础代码.
但如果说代码中有用到异步加载的话.则需要考虑到manifest的缓存问题的.也就是说manifest的hash需要对应上.

**解决方法**
解决manifest的hash不变的方法是使用HtmlWebpackInlineSourcePlugin
```javascript
plugins: [
    new WebpackMd5Hash(),
    new HtmlWebpackInlineSourcePlugin(),.
    new HtmlWebpackPlugin({
            filename: './dist/index.html',
            template: './index.html',
            inject: true,
            minify: {
                removeComments: true,
                collapseWhitespace: true,
                removeAttributeQuotes: true
            },
            inlineSource: 'manifest.*.js$', // 注意加上这一句
            chunks: ['manifest', 'vendor', 'app']
        }),
]
```
他的作用就是把manifest的内容直接嵌入到html里面,而不是通过script标签加载manifest.
那么这时候.manifest一旦改变就会直接映射到index.html的改变.
因为是直接嵌入到html里面的.这样又会带来个问题就是manifest不能被缓存.
但manifest大小只有1.5kb左右.在当前网络情况下影响不会很大.

**总结**
由webpack缓存引发的问题最终的解决方法就是使用webpackMd5Hash和HtmlWebpackInlineSourcePlugin两个插件.
当然,这不是最完美的解决方法.当确实当下我所想到最好的解决方法.
如果各位有更好的解决方法不妨给我说下哦.

**题外话**
解决了前端的缓存,其实服务器的缓存策略也有一定的影响.
可以看下我的上一篇文章.里面有提到服务器的缓存策略对前端的影响.
谢谢大家!