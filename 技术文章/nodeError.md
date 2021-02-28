#### 前言 
> 我们平时在做nodejs开发的时候，通常在生产环节，做异常监控，出现问题能够及时记录到日志、及时告警，所以对于异常的监控就显得格外的重要

#### 1. try catch
> 用于捕获代码运行时的同步错误，有个致命缺陷，不能监控到异步的代码
```
try {
    throw new Error('这是err message')
} catch(err) {
    console.error('已经收到error message：' + err)
}
```


#### 2. process.on('uncaughtException', cb)

> 在全局处理没有监控到的错误, 可以给try catch补漏，但是其实非常糟糕！！！uncaughtException不能显示错误上下文堆栈信息，出错了，也根本无法排查，那还上报个锤子。其次，一旦事件触发了，node进程会crash，极其不友好。
```
try {
    setTimeout(() => {
        throw new Error('这是err message')
    }, 0)

} catch(err) {
    // 根本就收不到
    console.log('已经收到error message：' + err)
}

process.on('uncaughtException', (err) => {
    // 收到了
    console.error('收到了没有监控到的异步错误', err);
});
```


#### 3. domain
> 用domain运行监控的代码, 注册error事件，可以捕获异步错误，显示详细的错误堆栈信息，不会触发uncaughtException让进程崩溃，极其好用！！！！

```
const domain = require('domain')
const dd  = domain.create();

dd.on('error', function(err) {
    console.error('捕捉到异常了', err);
});


dd.run(() => {
    setTimeout(() => {
        throw new Error('异步错误信息：12345')
    } , 0)
})

```

#### 4. async function

> 在用async await处理promise.then的时候，在函数内部无法捕获promise rejected状态的错误，但是在外部还是可以的，因为await相当于返回一个promise，通过这个这promise的catch就能捕获到错误了。

> 异步一定要封装在promise内部，即是不用reject方法也能捕获得到

```
async function test(){
    function request(){
        return new Promise(() => {
            throw new Error('error, promise')
        })
    }
    let res = await request()
}

test().catch((err) => {
    console.log('捕获到了错误', err)
})

```
