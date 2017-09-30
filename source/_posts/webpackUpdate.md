---
title: 记一次vue-cli中webpack升级2.x
date: "2017-2-17 13:30:47"
tags: [前端,Js, Vue, webpack]
---
项目初建是由vue-cli创建的.
vue-cli虽然搭建项目很方便，但同时也会增加复杂度
第一次用的时候我对那一堆文件是不明觉厉，不敢动的。
后来才慢慢了解各个文件用处。

webpack发布2.x版本好像也挺久了，刚好昨天下午事比较少，就打算挖个坑升级下。
看了下webpack2.x的升级指南，也就几个点需要更改，至于哪几个点这里就不说了，一搜一大堆。

**<------- 2.22有所更新,反省内容在最底部 -------->**

<!-- more -->

接下来当然就是更新webpack了
```
npm install webpack@2.1.1 webpack-dev-server@2.1.0-beta.10 extract-text-webpack-plugin@2.0.0-beta.4
```

更新完webpack，接着就对照着他的升级指南修改相应的配置了。

因为2.x中将resolve中的几个命令合并了
所以在原来webpack.base.conf.js中，修改
``` javascript
  resolve: {
    extensions: ['.js', '.vue', '.scss'],// 注意此处少了“”这个元素
    // fallback: [path.join(__dirname, '../node_modules')],  去掉即可
    modules: [  // 这是2.x中新增的
       path.join(__dirname, "src"),
       "node_modules"
    ]，
    alias: {
      // 你的别名配置
    }
  }，
  resolveLoader: {
    // fallback: [path.join(__dirname, '../node_modules')] 去掉即可
  },
  modules： {
    // preLoaders: [], webpack2,x中去掉此选项,改成了在rules中配置
    // webpack2.x中要求将loaders改成rulse,并且将原来的额外配置写入options就可以
    // 在这里之列出几个
    rules： [  
    {
        test: /\.vue$/,
        // loader: 'eslint-loader',
        use: [
          {
            loader: 'eslint-loader',
            // 不知道这里干嘛的看下面就清楚了
            options: {
              formatter: require('eslint-friendly-formatter')
            }
          }
        ],
        include: projectRoot,
        enforce: "pre", // 这两个是原来配置在preLoader中的,如今需要写在这里
        exclude: /node_modules/
      },
      {
        test: /\.js$/,
        // loader: 'eslint-loader',
        use: [
          {
            loader: 'eslint-loader',
            options: {
              formatter: require('eslint-friendly-formatter')
            }
          }
        ],
        include: projectRoot,
        enforce: "pre", // 这两个是原来配置在preLoader中的,如今需要写在这里
        exclude: /node_modules/
      },
      // 接下来的是原有的loader的,将loader按照最新的use写法即可,问题不大
      // 但是vue的配置可能需要改写

      // webpack1.x写法,还有额外的vue配置是写在下方的vue和sassLoader中,但2.x不支持
      //  {
      //   test: /\.vue$/,
      //   loader: 'vue'
      // },
      // webpack2.x写法
      {
        test: /\.vue$/,
        use: [
          {
            loader: "vue-loader",
            options: {
              // 这个vueCssLoader其实就是按照原来的cssLoaders改写的.
              loaders: utils.vueCssLoaders({
                            sourceMap: process.env.NODE_ENV === 'production' ? config.build.productionSourceMap : config.dev.cssSourceMap,
                            extract: process.env.NODE_ENV === 'production' ? config.build.extract : config.dev.extract
                        })
            }
          }
        ]
      },
      {
        test: /\.js$/,
        // loader: 'babel-loader',
        use: ['babel-loader'],
        include: projectRoot,
        exclude: /node_modules/
      },
      {
        test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
        // loader: 'url-loader',
        use: [
          {
            loader: 'url-loader',
            options: {
              limit: 10,
              name: utils.assetsPath('img/[name].[hash:7].[ext]')
            }
          }
        ]
      }
      // JSON loader默认自带,可去掉
      //{ 
      //  test: /\.json$/,
      //  loader: 'json'
      //},
    ]
  },
  // 下面这几个是原来webpack中配置的,webpack2.x中将去掉,改成loader中的options配置项
  // eslint: {
  //   formatter: require('eslint-friendly-formatter')
  // },
  // vue: {
  //   loaders: utils.cssLoaders()
  // },
  // sassLoader: {
  //   includePaths: [path.resolve(__dirname, '../src/sass')]
  // }
```

上面提到了我用到了一个vueCssLoader的函数是由原来的cssLoader改写的,下面就看下代码

**(2.22更新:不推荐使用了,理由在最下面,此段可以忽略)**

```javascript
/**
  cssLoaders就是为vue中的style生成loaders去解析不同语言编写的css
**/
exports.cssLoaders = function (options) {
  options = options || {}
  // generate loader string to be used with extract text plugin
  function generateLoaders (loaders) {
    var sourceLoader = loaders.map(function (loader) {
      var extraParamChar
      if (/\?/.test(loader)) {
        loader = loader.replace(/\?/, '-loader?')
        extraParamChar = '&'
      } else {
        loader = loader + '-loader'
        extraParamChar = '?'
      }
      return loader + (options.sourceMap ? extraParamChar + 'sourceMap' : '')
    }).join('!')

    // 如果需要单独打包成css文件
    if (options.extract) {
      // Creates an extracting loader from an existing loader. 
      // Supports loaders of type { loader: string; query: object }.
      // css样式没有被抽取的时候可以使用该loader
      return ExtractTextPlugin.extract('vue-style-loader', sourceLoader)
    } else {
      // 否则使用vue-style-loader 将css嵌入html中
      return ['vue-style-loader', sourceLoader].join('!')
    }
  }

  // http://vuejs.github.io/vue-loader/configurations/extract-css.html
  return {
    css: generateLoaders(['css']),
    postcss: generateLoaders(['css']),
    less: generateLoaders(['css', 'less']),
    sass: generateLoaders(['css', 'sass']),
    scss: generateLoaders(['css', 'sass']),
    stylus: generateLoaders(['css', 'stylus']),
    styl: generateLoaders(['css', 'stylus'])
  }
}

/**
  这个是用来生成webpack2.x支持的vue配置
  这里只根据是否需要单独打包成css来区分两种loader
  如果需要用到autoprefixer的话
  则在根目录创建 postcss.conf.js,
  写入即可
  module.exports = {
    plugins: [
      require('autoprefixer')({browsers: ['last 5 versions', 'Android >= 2.0']})
    ]
  }
**/
exports.vueCssLoaders = function(options = {}) {
    // 因为我使用sass,所以并不需要像cssLoaders那样兼顾所有,所以我只写了css,和sass,如果需要的,可以自己添加其他语言
    // 在这里配置的loaders并不和rules一样的配置,而是使用原有的配置方案.即通过!来连接loader
    const loaders = {
        sass: ['css-loader', 'postcss-loader', 'sass-loader?includePaths=' + [path.resolve(__dirname, '../src/sass')]],
        css: ['css-loader', 'postcss-loader']
    };
    let result = {}; //最终返回给vue的loader配置对象
    
    for (key in loaders) {
        const loader = loaders[key];
        // 是否需要sourceMap
        loader = loader.map((item) => {
            let extraParamChar = /\?/.test(item) ? '&' : '?';
            return item + (options.sourceMap ? extraParamChar + 'sourceMap' : '')
        })
        // 如果需要单独打包成css文件
        if (options.extract) {
            // Creates an extracting loader from an existing loader. 
            // Supports loaders of type { loader: string; query: object }.
            // css样式没有被抽取的时候可以使用该loader
            // return ExtractTextPlugin.extract('vue-style-loader', loader)
            result[key] = ExtractTextPlugin.extract({
                fallback: 'vue-style-loader',
                use: loader.join('!')
            })
        } else {
            // 否则使用vue-style-loader 将css嵌入html中
            result[key] = ['vue-style-loader', loader.join('!')].join('!')
        }

    }
    return result;
}
```

到这里是不是就可以直接运行了呢,其实并不能.webpack.base.conf.js是配好了,但是webpack.dev.conf.js并没有配好.

在webpack.dev.conf.js中,只需要改两处地方
第一处是module
```javascript
module: {
  rules: utils.styleLoaders({ sourceMap: config.dev.cssSourceMap, extract: false }) 
}
```
第二处是NoErrorsPlugins
```javascript
plugins: [
    new webpack.NoEmitOnErrorsPlugin() // 替代了原来的NoErrorsPlugins
  ]
```

你以为这样就好了? 并不是,因为utils.styleLoaders已经不适合webpack2.x语法了,因此需要修改.

其实可以直接去配置相应环境的loader,但是我为了方便,还是写个简单的方法来生成loader.这里只根据是否需要单独打包成css来区分两种loader

**(2.22更新:不推荐使用了,理由在最下面,此段可以忽略)**
```javascript
exports.styleLoaders = function(options = {}) {
    // 用来生成loader
    const extract = function(loaders) {
      // 是否需要打包成css
      if(options.extract) {
        loaders = loaders.map(loader => {
          let result = '';
          if(typeof loader === 'string') {
            result = loader;
          } else {
            result = loader.loader;
            if(loader.options) {
              result += '?';
              for(key in loader.options) {
                result += key + '=' + loader.options[key] + '&'
              }
            }
          }
          return result;
        }).splice(1).join('!') //splice是为了把style-loader去掉,因为在ExtractTextPlugin中fallback会用到style-loader
        return ExtractTextPlugin.extract({
                  fallback: 'vue-style-loader',
                  use: loaders
              })
      }
      // 不需要则原路返回loaders
      return loaders;
    }
    return [
        {
            test: /\.css$/,
            use: extract([ //调用extract函数来生成对应的loaders
                'vue-style-loader',
                {
                    loader: 'css-loader',
                    options: {
                        sourceMap: options.sourceMap || false,
                        minimize: true
                    }
                },
                'postcss-loader',
            ])
        },
        {
            test: /\.scss$/,
            use: extract([
                'vue-style-loader',
                {
                    loader: 'css-loader',
                    options: {
                        sourceMap: options.sourceMap || false,
                        minimize: true
                    }
                },
                'postcss-loader',
                {
                    loader: 'sass-loader',
                    options: {
                        includePaths: [path.resolve(__dirname, '../src/sass')],
                        sourceMap: options.sourceMap || false
                    }
                }
            ])
        }
    ]
}
```

webpack.dev.conf.js改好了.这时候可以运行开发环境了,但如果想要运行生产环境的话,还要改下webpack.prod.conf.js
```javascript
  module: {
    rules: utils.styleLoaders({ sourceMap: config.build.productionSourceMap, extrack: true })
  },
  plugins: [
    new ExtractTextPlugin({
      filename: utils.assetsPath('css/[name].[contenthash].css'),
      allChunks: false
    })
  ]
```

改到这里可以算是升级完成了
如果后续还发现其他坑,会继续补上!


## # 02.22 更新
今天才发现尤大大写过相关的配置
[https://github.com/vuejs-templates/webpack/tree/master/template/build](https://github.com/vuejs-templates/webpack/tree/master/template/build)。

(最后发现..原来cssLoader和styleLoader不用怎么改都可以跑起来.真是图样图森破...不过自己折腾一下也没坏处.各位看下就好别太认真了.)

看完之后对比我自己写的

发现....
其实我所写的两个loaders方法实际是仿照webpack1.x的语法来改写从而兼容webpack2.x的loader

而尤大大是按照webpack2.0语法来写的

所以还是尤大大的比较合适.

但我后面又改了下...加了postcss的autoprefixer.

不然的话只有通过vue-loader解析的样式才会加上前缀,而直接通过import进来的样式并不会添加前缀


其实就是在cssLoader中添加postcss而已..

```javascript
exports.cssLoaders = function(options = {}) {
    ...
    var cssLoader = {
        loader: 'css-loader',
        options: {
            minimize: true,
            sourceMap: options.sourceMap
        }
    },
    postcssLoader = {
        loader: 'postcss-loader'  // 加了这个
    }

    function generateLoaders (loader, loaderOptions) {
        var loaders = [cssLoader, postcssLoader]  // 加了postcssLoader
        ...
    }
```

好了.自我反省完毕.!