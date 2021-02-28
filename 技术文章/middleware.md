#### 前言
> koa2 和 express4都是非常优秀的nodejs http服务框架。koa2做到了小而美，十分轻量化。express做得很完善，大而全。

> 这两种框架在处理http请求响应上下文的时候，都采用了中间件模式。单纯从技术角度看，设计的非常巧妙，运用了纯函数、颗粒化、尾递归能高级技巧。

> 所以今天从源码角度来分析一下其巧妙之处。并实现一下Koa2的核心功能，简单写一个koa！



#### nodejs中间件

> 就是一个函数，参数是请求、响应的上下文。在接口返回响应之前，做一系列处理；

#### compose合成函数

> compose是合成的意思，就是让很多中间件组成的数组合成一个颗粒化的函数，何时执行下一个中间件由当前中间件控制。它有些像递归，可以控制什么时机进行下一次的调用。

#### koa-compose

源码
```
function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

> 赞眼一看非常复杂，但是可以通过语义话变量进行精简, 先来看一下compose是怎么使用的。


```
// koa内部
koaCompose([中间件函数1， 中间件函数2])(context)

// 使用时候

app.use(async (context, next) => {
    console.log('start')
    await next()
})

```

> koa采用的是洋葱模型，每次执行await next(), 会阻塞中间件内部后面的代码执行，等结下来的中间件都执行完，再回过来执行next下面的代码


等价的代码
```
function myCompose(middlewares = []) {
    let lessOneThanIndex = -1;
    return function(ctx){
        function dispatch(i){
            if (lessOneThanIndex !== i - 1) {
                throw new Error('next执行了两次')
            } else {
                lessOneThanIndex = i
            }
            if (i === middlewares.length) {
                // 越界了
                return Promise.resolve()
            }
            let middleware = middlewares[i]
            try {
                let nextFn = dispatch.bind(null, i + 1)
                let res = middleware(ctx, nextFn)
                return Promise.resolve(res)
            } catch(e) {
                return Promise.reject(e)
            }
            
        }
        return dispatch(0)
    }
}
```

> dispatch主要做了5件事
1. 按次序将每个中间件从数组取出； 
2. 将下一次dispatch的执行函数作为next，
3. 像中间件函数传入next参数并调用
4. 像中间件执行结果以promise形式返回 (await next()想当于 await dispatch(n))
5. 不能让中间件的next执行两次。


> koa-compose是koa的核心，接下来可以实现一个简单版的koa，包含异常处理

```
const http = require('http')
const EventEmitter = require('events').EventEmitter; 

function koaCompose (middlewares = []) {
    return function (context, next) {
        let index = -1;
        function dispatch (i) {
        //   if (i <= index) return Promise.reject(new Error('next() called multiple times'))
            index = i           // index唯一作用是防止中间件内部的next函数执行两次, i应该永远只比index大1
            let fn = middlewares[i]  // 第一次调用，第一个中间件函数
            if (i === middlewares.length) {
                // i已经越界了，next是undefined
                fn = next
                return Promise.resolve()
            }
            try {
                // 将下个中间件作为next参数
                let nextFn = dispatch.bind(null, i + 1);    // next其实是处理下一个中间件的，dispatch函数，
                // 将当前中间件的运行结果以promise形式返回
                let runCurrentMiddleware = fn(context, nextFn)
                return Promise.resolve(runCurrentMiddleware);
            } 
            catch (err) {
                return Promise.reject(err)
            }
        }
        return dispatch(0)
    }
}

async function mid1(ctx, next){
  console.log('1 start')
  await next()
  console.log('1 end')
}

async function mid2(ctx, next){
    ctx.haomin = 123456;
    throw new Error('mid2Error')
    await next()    // 等第二个中间件fulfilled后在执行下面逻辑
    console.log('2 end')
}


class Koa extends EventEmitter {
    constructor(){
        super()
        this.middlewareList = []

    }
    use(midd){
      this.middlewareList.push(midd)
    }
    listen(...args){
        http.createServer((req, res) => {
            if (req.url === "/favicon.ico") return 
            const ctx = {
                req,
                res
            }
            koaCompose(this.middlewareList)(ctx)
            .catch((err) => {
                this.emit('error', err, ctx)
            })
            res.end('hello123')

        }).listen(...args);
    }
}

const app = new Koa()

app.on('error', (error, ctx) => {
    console.log('上报error', error, ctx)
})

app.use(mid1)
app.use(mid2)
app.listen(3000)
```

> 主要方法有三个，use方法注册中间件, compose将ctx参数和next方法注入到每个中间件的参数中，并让第一个中间件开始执行，on方法继承event，用于监控中间件内部的错误，只能监控同步的，如果是异步的错误，必须封装成promise，再用await，这就使中间件函数有一个promise的返回值，最后可以通过catch监听到，再用emit派发。so easy！


#### express的compose
> express也是洋葱模型，但没有使用async function，在中间件内部处理异步就麻烦很多，而且捕获异常也很麻烦。

```
const mid1 = (req, res, next) => {
    console.log('mid1 start')
    next()
    console.log('mid1 end')
}

const mid2 = (req, res, next) => {
    console.log('mid2 start')
    next()
    console.log('mid2 end')
}

const expressCompose = (middlewares = []) => {
    return function(req, res){
        let next = () => {
            let middleware = middlewares.shift();
            if (!middleware) {
                return;
            }
            return middleware(req, res, next)
        }
        next()
    }
}

expressCompose([mid1, mid2])('reqest对象', 'response对象')

```

> express的compose很粗暴，直接将next函数作为中间件区执行，知道中间件队列消耗完。


#### 总结

nodejs中间件看似很复杂，其实很简单。其实现的核心主要有以下几点：
1. use注册方法，用于将中间件函数加入到中间件队列；

2. compose方法，将中间件队列整合，注入上下文对象，注入next方法吗用于控制中间件执行。将第一个中间件执行。
