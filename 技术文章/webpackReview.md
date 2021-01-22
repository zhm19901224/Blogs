## 回顾：webpack 4.X - 5
### webpack 模块化打包工具


#### npx介绍
> npm5.2开始，就有了npx。npx是node包执行工具。

> npm 包名

> npm默认行为，在项目录下node_modules查找是否有软件包，如果有就执行，如果没有就去下载最新版本再执行
> 
> npm 相当于进入node_modules/bin/my_package/pacakge.json找到script，再找到script执行，或者npm run myPackage  现在只需要npx mypackage就可以了

#### 初始化工程环境
touch webpack-study
cd webpack-study
npm init
npm info webpack    查看历史稳定版本
npm i webpack webpack-cli url-loader style-loader css-loader sass-loader node-sass postcss-loader autoprefixer -D

#### 基本配置

##### 1. mode
> mode为development，打包出来的脚本会被压缩

##### 2. entry、output
```
output: {
    chunkFilename: '[name].chunk.js', // 异步加载的js形成chunk文件，为其命名
}

```


##### 3. module 配置如何用不同loader处理不同类型模块

###### 3.1 url-load处理图片资源
> 1. 如果图片大小小于2k， 打包成base64很好
```
{
    test: /\.(jpe?g|png|gif)$/,
    use: {
        loader: 'url-loader',
        options: {
            name: '[name]_[hash].[ext]',
            outputPath: 'images/',
            limit: 2048
        }
    }
}
```

###### 3.2 处理css、sass、less、stylus等样式表文件

> 1. css-loader 分析引入的css文件内部还有哪些引用的其他样式，合并成一个css文件
```
{
    test: /\.css$/,
    use: ['style-loader', 'css-loader']
}
```
> 2. style-loader 将样式以style属性方式插入到标签内，css in js形式

> 3. 处理sass文件
```
// npm i node-sass sass-loader -D
{
    test: /\.(sass|scss)$/,
    use: ['style-loader', 'css-loader', 'sass-loader']
}
``` 
> 4. 处理css3属性的兼容前缀
```
// npm i postcss-loader autoprefixer -D
// 在项目目录下创建postcss.config.js
module.exports = {
    plugins: [
        require('autoprefixer')
    ]
}

// webpack.config.js
{
    test: /\.(sass|scss)$/,
    use: ['style-loader', {
        loader: 'css-loader',
// 对于sass文件内部引入的sass或者css也需要先走前面的postcss、sass-loader两个loader先处理。非常重要！
        options: {
            importLoaders: 2
        }
    }, 'sass-loader', 'postcss-loader']
}

```

> 5. css模块化

> 让一个js文件中引入的样式文件只作用于当前js文件的dom节点
```
webpack.config.js
{
    loader: 'css-loader',
    options: {
        importLoaders: 2,
        modules: true
    }
}

// index.js
import style from './index.scss'
// style是所有class的集合对象
```

> 6. 将css文件以head中的link方式引入，而不再使用css in js
```
// npm i mini-css-extract-plugin
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
// module.rules
{
    test: /\.css$/,
    use: [MiniCssExtractPlugin.loader, 'css-loader', 'postcss-loader']
},


plugins: [
    new MiniCssExtractPlugin({
        filename: '[name].css',
        chunkFilename: '[name].chunk.css'
    })
]
```

> 7. 压缩css
```
// npm i css-minimizer-webpack-plugin --save-dev
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

optimization: {
    splitChunks: {
        chunks: 'all',
    },
    minimize: true,
    minimizer: [
        new CssMinimizerPlugin({
            test: /\.css$/,
        }),
    ],
},
```
###### 3.3 file-loader处理字体文件（只是移动字体图标到dist下）
```
{
    test: /\.(eot|ttf|svg)$/,
    use: {
        loader: 'file-loader',
    }
}
```


#### 4. plugins

##### 1. html-webpack-plugin
> 根据模版（可以没有）在打包时，在打包目录下生成html，并把打包后的js、css自动引入
```
const HtmlWebpackPlugin = require('html-webpack-plugin');

plugins: [
    new HtmlWebpackPlugin({
        template: 'src/index.html',
        minify: true        // 压缩
    })
]
```

##### 2. html-webpack-plugin
> 打包前，先删除打包目录内的内容
```
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
plugins: [
    new CleanWebpackPlugin(),
    new HtmlWebpackPlugin({
        template: 'src/index.html',
        minify: true        // 压缩
    })
]
```

##### 3. webpack.HotModuleReplacementPlugin

#### 如果html放自己服务器，而js上传cdn，那么html需要引入的js前面需要有域名前缀, 需要设置output的publicPath

```
    output: {
        publicPath: 'http://qiniuyun.com/space/haomin/',
        filename: 'bundle.js',
        path: path.resolve(__dirname, 'dist')
    },

```


#### 5. sourcemap
> mode如果是development，会自动开启sourcemap。devtool： false关闭sourcemap

> inline: 打包后不会输出大度的.map映射文件。将映射欢喜直接以base64方式挂在业务代码上。打包快了，但导致业务代码体积变大。

> cheap：打包出的bundle的错误提示，只会精确到某个文件的某一行，不精确到第几个字符。忽略掉第三方模块内部的错误

> eval 打包最快，用eval方式执行映射关系。

> module： 不仅只管业务代码, 从node_modules中引入的第三方模块也会进行报错提示

最佳实践：development环境用eval-cheap-module-source-map
production环境用cheap-module-source-map


#### 6. webpack-dev-server
> watch业务脚本的变化，有变化重新打包
##### 1. webpack --watch

##### 2. devServer配置
```
// npm i webpack-dev-server -D
devServer: {
    contentBase: path.join(__dirname, 'dist'),  // 服务器路径
    compress: true,     // 开启gzip压缩
    port: 9000
}

// package.json cli配置
script: {
    "dev": "webpack serve"      // webpack5的写法，4.X是webpack-dev-server
}
```

##### 3. 自定义服务，使用webpackDevMiddleware中间件去监管文件变化
```
// npm i express webpack-dev-middleware -D
// touch server.js   
const webpack = require('webpack')

const express = require('express')

const webpackDevMiddeware = require('webpack-dev-middleware');

const webpackConfig = require('./webpack.config.js')

const compiler = webpack(webpackConfig);        // 根据配置文件生成的编译器

const app = express()

app.use(webpackDevMiddeware(compiler, {
    publicPath: webpackConfig.output.publicPath     //  只要文件发生改变，就重新编译，输出在配置文件的publicPath目录下
}))

app.listen(3000, () => {
    console.log('server is running')
})

```


#### 7. HMR 热模块替换
> CSS HMR：如果样式文件发生变化，页面上的元素的状态（数量）不会改变，只是样式改变。只有脚本变化才会重新编译。css-loader底层已经实现，无需写脚本
```
    devServer: {
        contentBase: path.join(__dirname, 'dist'),
        compress: true,
        port: 9000,
        open: true,
        hot: true
    },
    plugins: [
        new webpack.HotModuleReplacementPlugin()
    ]
```
> JS HMR：如果某一个子模块发生改变，只重新编译该模块的内容，其他模块在页面的状态不会改变，vue、react借助babel-preset，内置了此功能，无需写脚本。如果不用框架，需要自己手动实现

> 如果是一些第三方库动态渲染，如数据图表渲染库echarts什么的，需要自己去写hmr代码

> HMR 原理https://webpack.js.org/concepts/hot-module-replacement/
```
    // webpack.config.js
    devServer: {
        contentBase: path.join(__dirname, 'dist'),
        compress: true,
        port: 9000,
        open: true,
        hot: true
    },
    plugins: [
        new webpack.HotModuleReplacementPlugin()
    ]
    // 入口index.js
    import sonA from './sonA'   // sonA是一个模块内的函数
    import sonB from './sonB'
    if (module.hot) {
        // 哪个模块文件发生变化，就重新执行该模块的渲染函数
        module.hot.accept('./sonA', () = > {
            sonA()  
        })
        module.hot.accept('./sonB', () = > {
            sonB()
        })
    }
```

#### 8. babel处理es2015+ javascript脚本
> @babel/core: babel核心库，用于将源码转成ast，再将ast转其他内容的工具集合

> babel-loader: webpack和babel关联的桥梁，核心功能不在内部

> @babel/preset-env 内置转译成es5的规则

> @babel/polyfill 垫片，补充低版本浏览器内部的全局变量，比如promise、map。。。babel7以后，已经不推荐用了，babelcore已经内置了。这东西打包出来的代码很大，很多方法如果没有使用到，就很浪费了。可以通过preset-env的useBuiltIns: 'usage'配置，根据具体使用了哪些包来决定打包哪些部分的polifill。如果开发组件库或者类库，会污染全局环境，不适合用。

> @babel/runtime @babel/plugin-transform-runtime
> 写库的时候要这么用，不要用polifill。runtime会以闭包和别名形式注入api
```
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "absoluteRuntime": false,
        "corejs": false,
        "helpers": true,
        "regenerator": true,
        "useESModules": false,
        "version": "7.0.0-beta.0"
      }
    ]
  ]
}
```

> npm install --save-dev @babel/plugin-syntax-dynamic-import 
> 可以写异步import的实验性语法
#### 9. 处理react的jsx文件
> npm install --save-dev @babel/preset-react

```
// .babelrc preset有顺序，从后往前转换，先转换react的jsx到createElement，再转换es6到es5
{
    "presets": [
        [ "@babel/preset-env", {
            "targets": {
                "chrome": "67" 
            },
            "useBuiltIns": "usage"
        }],
        "@babel/preset-react"
    ]
}
```
#### 10. tree-shaking
> 引用一个模块中导出的变量，如果有一些没引入的，不会打包。只有ESM才支持tree-shaking

> tree-shaking必须配置副作用白名单，否则样式表失效，后果严重！！！！
```
// package.json
  "sideEffects": [ "*.css", "*.scss", "*.less", "*.stylus", "babel-polyfill"],

// 不这么做，没有导入变量的导入，全都被摇掉了，就惨了！！！！
```

> 思考？ webpack和rollup都用到了tree-shaking，但rollup的打包体积更小，为什么？

首先webpack打包后会生成__webpack_require__等runtime代码，这就会占用一定的体积。而rollup不存在这个问题。
Tree-shaking的原理是通过ES Module的方式进行静态分析，同时对程序流进行分析，判断变量是否被使用。之后就是比对二者Tree-Shaking的优劣：二者都是通过ES Module的方法进行静态分析，但rollup有程序流分析的功能，可以更好地判断代码是否有副作用，从而进行更彻底的Tree-shaking；而webpack是在production模式中通过uglifyJS进行Tree-Shaking，对于程序流的分析不完善，所以依然会有无法删除的代码。

> 如果打包js库，给使用者用，最好用rollup因为rollup可以导出umd和esm，esm可以进行tree-shaking。而且最好一个方法一个文件。类似antd。使用时可以
```
import Button from 'antd/lib/button';

// 设计了babel-pluigin后可以将
import { Button, Message } form 'antd'转化成上面形式
```


#### 11. 分环境打包
> npm i webpack-merge -D

#### 12. code-spliting
> all 不论同步还是异步代码，只要是node_modules里面的第三方代码，就会被分割成单独的vendor文件

```
// webpack.base.js
optimization: {
    splitChunks: {
        chunks: 'all',
    },
},

// 支持异步加载模块，并为异步加载的第三方模块打包后命名
npm i @babel/plugin-syntax-dynamic-import -D

// .babelrc
"plugins": ["@babel/plugin-syntax-dynamic-import"]

// 业务代码index.js 魔法注释
import(/* webpackChunkName: "lodash"*/ 'lodash').then()
```

> splitChunkPlugin配置参数作用

```


    optimization: {
        splitChunks: {
          chunks: 'all',   // 不论同步引入模块，还是异步引入，都做代码分割  （异步加载路由页面）
          minSize: 20000,   //  只有引入大于20k的库才被代码分割
          minRemainingSize: 0,
          minChunks: 1,         // 如果异步加载3个组件，这3个组件，形成了3个chunk，这三个chunk都引入了lodash，那么lodash才会被分割进入vendor.js
          maxAsyncRequests: 5,     // 打包前五个第三方库，会做代码分割，都打包进vendor，后面的就不会了这么做了(因为并行加载有上线)
          maxInitialRequests: 3,   // 入口文件如果引入6个js，只对前三个做代码分割
          enforceSizeThreshold: 50000,
          cacheGroups: {            //  
            defaultVendors: {
              test: /[\\/]node_modules[\\/]/,
              priority: -10,            // 优先级，放在node_module第三方库的优先级很高，会被打包进vendor.js
              reuseExistingChunk: true,
              filename: 'vendor.js'
            },
            default: {
              minChunks: 2,         // 只有引入至少两个自己的业务模块，才会被打包进
              priority: -20,
              reuseExistingChunk: true,
              filename: 'common.js'     //
            }
          }
        }
      }
```


#### 13. 组件的懒加载与预加载 preload、prefetch

> npm i @babel/plugin-syntax-dynamic-import -D 对异步函数的支持

> 只要使用了异步函数加载第三方库，第三方库的代码就会被分割成单独vendor文件。vue的组件懒加载就用了这个原理

> 浏览器控制台command shift p，show cavarage查看代码利用率

> 使用异步函数加载首屏用不到的模块，性能会非常好
```
import(/* webpackPrefetch: true */'./othenrPages.jsx')

// prefetch会在主流程加载完带宽空闲执行，preload会随主流程一起加载
```


#### 14. 打包分析
// http://webpack.github.com/analyse


#### 15. 缓存副作用
> 消除打包出来的模块名被长久缓存，开发修改脚本重新打包名字相同，导致协商缓存人文文件没变的问题。
```
// 每次文件发生改变，内容摘要生成hash作为新的文件名，这样不会产生缓存副作用
    output: {
        publicPath: '/',
        filename: '[name].[contenthash].js',
        chunkFilename: '[name].[contenthash].chunk.js',
        path: path.resolve(__dirname, '../dist')
    },
```


#### 16. 打包一个库(不如rollup)并发包

```
// github新建一个node类型项目，克隆到本地
// npm init
// npm login
// npm i webpack webpack-cli -D
// webpack.config.js

const path = require('path')
module.exports = {
    mode: 'production',
    entry: {
        index: './src/index.js'
    },
    output: {
        path: path.resolve(__dirname, 'dist'),
        filename: '[name].js',
        library: 'hm',   // 如果以script标签引入，会生成hm变量，为库的内容
        libraryTarget: 'umd'   // 支持amd、cjs、esm方式引入
    }
}

// npm set registry https://registry.npmjs.org/
// npm publish 发包
```

#### 17. Typescript打包

> npm i ts-loader typescript -D

> 根据一定规则确定一个tsconfig.json

> 用ts-loader处理ts文件
```
// module.rules
{
    test: /\.tsx?$/,
    use: 'ts-loader',
    exclude: /node_modules/
},
```

#### 18. 配置eslint
> npm i eslint -D
> npx eslint --init

#### 19. 打包调优
1. 配置loader时，用include、exclude缩小load处理范围，对于第三方库，不再做babel处理，因为本身已经都是处理过的
2. 精简plugin，比如在development环境下不做任何代码压缩。
3. resolve配置参数。如：导入文件扩展名自动判断补全、别名, 不能配太多，否则dev环境文件查找损耗性能。只针对逻辑文件，可以配置
4. 使用dllplugin提高dev环境打包速度
> 第三方的模块，通常都是不会变的，所以把他们打包到一个文件夹下，第一次打包只分析一次，以后再次打包就复用分析好的结果。

> webpack.DllPlugin 根据打包的库文件变量，和真正的库，生成映射关系.json文件

> add-asset-html-webpack-plugin 用于引入打包后的静态资源，到html中

> webpack.DllReferencePlugin 用于分析库映射文件，来决定是打包node_modules里面的库，还是直接从dll文件夹内，打包出来的库文件变量直接引入

#### 20. 工程实战
> https://github.com/zhm19901224/hm-common-template