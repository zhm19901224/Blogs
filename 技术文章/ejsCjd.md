#### EJS和CJS是什么？
EJS: ES module
CJS: node中的commonJS
两者都是模块化方案，但彼此互不兼容。

#### EJS和CJS区别？
1. EJS导出的是变量或常量的引用，只有getter，没有setter接口，是只读的。重新赋值会报错。CJS导出的是复制后的副本
2. EJS中默认顶层"use strict"使用了严格模式，this是undefined, 而CJS中,this指的是当前模块。
3. EJS在打包器进行静态分析时就已经形成了，也叫编译输出。CJS时代码运行时同步加载。
4. CJS加载后，模块会被缓存。ES6模块是动态引用，并且不会缓存值，模块里面的变量绑定其所在的模块。


#### nodeJS中使用EJS的2种办法。
1. 将模块文件必须定义成.ejs结尾。然后使用import()函数加载。
2. 在其目录下建立package.json 里面写{ type: "module" }

#### EJS中使用CJS
es模块中可以加载cjs，但是只能整体加载
```
import A from './b.js';
```
因为CJS的module.export=xxx被视为一个整体，ejs静态分析不出里面内容

参考：阮一峰老师的博客http://www.ruanyifeng.com/blog/2020/08/how-nodejs-use-es6-module.html