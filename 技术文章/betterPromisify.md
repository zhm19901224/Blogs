#### 前言
总所周知，node8.X发布之后有了对async function的完美支持，util模块又添加了转换成promise的神奇util.promisify, 处理异步对比以前变得更加优雅了
> 之前做法
```
// befor:
const fs = require('fs')

fs.readFile('./abc.js', 'utf8', (err) => {
    if (!err) { 
        //... 
    }
})
```
> 现在做法
```
// after:
const fs = require('fs')
const promisify = require('util').promisify
const fsReadFile = promisify(fs.readFile)

const readAction = async () => {
    try {
        let res = await fsReadFile('./abc.js', 'utf8')
        // ...
    } 
    catch(err) {

    }
}
```
> 那么问题来了，用promise + async function处理了回调地狱，但是在日常开发中，我们使用的大部分node模块api都遵循Error Callback First。使用的时候也需要先进行异常的拦截。现在的做法明显不满足Error Callback First原则，而且try catch就不是嵌套吗？照样污染了我们的代码。


#### 解决方案
> 我们期望能够遵照Error Callback First写业务，并且像golang一样优雅，不写那么多try catch, 就像这样
```
const fs = require('fs')
const promisify = require('util').promisify
const fsReadFile = promisify(fs.readFile)

const readAction = async () => {
    let [err, content] = await fsReadFile('./abc.js', 'utf8')
    if (!err) {
        // 操作content内容
    }
}
```

#### 利用柯里化思想，实现一个betterPromisify
```
const betterPromisify = (fn) => {
    return function(...args) {
        return new Promise((resolve, reject) => {
            promisify(fn)(...args)
            .then((res) => {
                resolve([null, res])
            })
            .catch((err) => {
                resolve([err])
            })
        })
        
    }
}
// const fsReadFile = betterPromisify(fs.readFile);
```

#### 总结
> 不能总依赖第三方模块，自己觉得用的爽才是硬道理，用的不爽就再造一个爽的！