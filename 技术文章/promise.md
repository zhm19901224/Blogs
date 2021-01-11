### 先了解一下promise A+
> 规范原文地址：http://promisesaplus.com

#### 1. 术语
promise是一个包含了兼容promise规范then方法的对象或函数，thenable 是一个包含了then方法的对象或函数。value 是任何Javascript值。 (包括 undefined, thenable, promise等).exception 是由throw表达式抛出来的值。reason 是一个用于描述Promise被拒绝原因的值。
#### 2. 要求
##### 2.1 Promise状态
一个Promise必须处在其中之一的状态：pending, fulfilled 或 rejected.
如果是pending状态,则promise：可以转换到fulfilled或rejected状态。如果是fulfilled状态,则promise：不能转换成任何其它状态。必须有一个值，且这个值不能被改变。如果是rejected状态,则promise可以：不能转换成任何其它状态。必须有一个原因，且这个值不能被改变。”值不能被改变”指的是其identity不能被改变，而不是指其成员内容不能被改变。
##### 2.2 then 方法
一个Promise必须提供一个then方法来获取其值或原因。Promise的then方法接受两个参数：
> promise.then(onFulfilled, onRejected)

1. onFulfilled 和 onRejected 都是可选参数：如果onFulfilled不是一个函数，则忽略之。如果onRejected不是一个函数，则忽略之。


2. 如果onFulfilled是一个函数:它必须在promise fulfilled后调用， 且promise的value为其第一个参数。它不能在promise fulfilled前调用。不能被多次调用。

3. 如果onRejected是一个函数,它必须在promise rejected后调用， 且promise的reason为其第一个参数。它不能在promise rejected前调用。不能被多次调用。


4. onFulfilled 和 onRejected 只允许在 execution context 栈仅包含平台代码时运行. 

5. onFulfilled 和 onRejected 必须被当做函数调用 (i.e. 即函数体内的 this 为undefined). 

6. 对于一个promise，它的then方法可以调用多次.当promise fulfilled后，所有onFulfilled都必须按照其注册顺序执行。当promise rejected后，所有OnRejected都必须按照其注册顺序执行。

7. then 必须返回一个promise
> promise2 = promise1.then(onFulfilled, onRejected);

> 如果onFulfilled 或 onRejected 返回了值x, 则执行Promise 解析流程[[Resolve]](promise2, x).

> 如果onFulfilled 或 onRejected抛出了异常e, 则promise2应当以e为reason被拒绝。

> 如果 onFulfilled 不是一个函数且promise1已经fulfilled，则promise2必须以promise1的值fulfilled.如果 OnReject 不是一个函数且promise1已经rejected, 则promise2必须以相同的reason被拒绝.
##### 2.3 Promise解析过程
Promise解析过程 是以一个promise和一个值做为参数的抽象过程，可表示为[[Resolve]](promise, x). 过程如下；
1. 如果promise 和 x 指向相同的值, 使用 TypeError做为原因将promise拒绝。
2. 如果 x 是一个promise, 采用其状态 
    1. 如果x是pending状态，promise必须保持pending走到x fulfilled或rejected.
    2. 如果x是fulfilled状态，将x的值用于fulfill promise.
    3. 如果x是rejected状态, 将x的原因用于reject promise.
3. 如果x是一个对象或一个函数：
    1. 将 then 赋为 x.then
    2. 如果在取x.then值时抛出了异常，则以这个异常做为原因将promise拒绝。
    3. 如果 then 是一个函数， 以x为this调用then函数， 且第一个参数是resolvePromise，第二个参数是rejectPromise，且：
        1. 当 resolvePromise 被以 y为参数调用, 执行 [[Resolve]](promise, y).
        2. 当 rejectPromise 被以 r 为参数调用, 则以r为原因将promise拒绝。
        3. 如果 resolvePromise 和 rejectPromise 都被调用了，或者被调用了多次，则只第一次有效，后面的忽略。
        4. 如果在调用then时抛出了异常，则：
            1. 如果 resolvePromise 或 rejectPromise 已经被调用了，则忽略它。
            2. 否则, 以e为reason将 promise 拒绝。如果 then不是一个函数，则 以x为值fulfill promise。
    
4. 如果 x 不是对象也不是函数，则以x为值 fulfill promise。


### 个人对该规范解读
#### 1. promise内部状态
> 三种状态pending、fulfilled、rejected，只要成为后两种任意一个，就不能改变

> fulfilled、rejected两种互斥的状态，只能变成其中一种

#### 2. then方法的硬性要求
> then方法接受两个可选的函数类型参数

> then方法内部既可以处理异步的任务，也可以处理同步的任务，需要通过state状态去判断

> then的返回值是一个promise对象，该promise的状态是什么？then的value值是什么？需要通过用户传入的onfulfilled回调函数的返回值具体去分析。



### 根据Promise A+规范，实现一个Promise类
#### 1. 先写一个只支持同步执行的promise。
```
class myPromise {
    constructor(executor){
        // 立即执行
        this.value = undefined;
        this.err = undefined;
        this.state = 'pending'  
        executor(this.onResolved.bind(this), this.onRejected.bind(this));
  
    }

    onResolved(value) {
        if (this.state === 'pending') {
            this.state = 'fulfilled'
            this.value = value
        }
    }

    onRejected(err) {
        if (this.state === 'pending') {
            this.state = 'rejected'
            this.err = err;
        }
    }

    then(onresolved, onrejected){
        if (this.state === 'fulfilled') onresolved(this.value)
        if (this.state === 'rejected') onrejected(this.err)
    }
}


let p = new myPromise((resolve) => {
    resolve('ok')
})
p.then((res) => {
    console.log(res)    // ok
})
```

> 对于执行异步的任务，比如
```
let p = new myPromise((resolve) => {
    setTimeout(() => resolve('ok'), 1000);
    
})
```
> 明显是不可以的，因为then的执行的时候此时state还是pending状态，而then只能执行状态是fulfilled与rejected的回调函数。所以，我们需要判断then要处理的是同步任务还是异步任务。如果then内部状态是pending，可以认为是异步任务，需要借助发布订阅模式，将onfulfiled、onrejected两种类型的回调函数先进行订阅，等真正执行resolve的时候，改变状态并将订阅的函数依次执行。来看一下既支持异步任务，又支持同步任务的promise: 

```
class myPromise {
    constructor(executor){
        // 立即执行
        this.value = undefined;
        this.err = undefined;
        this.state = 'pending'  
        this.onFulfilledFns = [];
        this.onRejectedFns = [];
        executor(this.onResolved.bind(this), this.onRejected.bind(this));
    }

    onResolved(value) {
        if (this.state === 'pending') {
            
            this.state = 'fulfilled'
            this.value = value
            this.resolveQueue.forEach((fn) => fn(this.value))

        }
    }

    onRejected(err) {
        if (this.state === 'pending') {
            this.state = 'rejected'
            this.err = err;
            this.rejectQueue.forEach((fn) => fn(this.err))
        }
    }
    then(onresolved, onrejected){
        if (this.state === 'pending' && onresolved) {
            // 异步情况，先不执行，先订阅，等真正resolve执行了，再发布
            this.onFulfilledFns.push(onresolved);
        }
        if (this.state === 'pending' && onrejected) {
            this.onRejectedFns.push(onrejected);
        }
        // 以下是同步的情况, 直接将cb直接执行并传入value就行了
        if (this.state === 'fulfilled') {
            onresolved(this.value)
        }
        if (this.state === 'rejected') {
            onrejected(this.err);
        }
    }
}
```

> 此时同步、异步任务都可以正确处理，现在解决最复杂的问题，如何进行链式调用。




```
class myPromise {
    constructor(executor){
        this.value = undefined;
        this.err = undefined;
        this.state = 'pending'  
        this.onFulfilledFns = [];
        this.onRejectedFns = [];
        executor(this.onResolved.bind(this), this.onRejected.bind(this));
  
    }

    onResolved(value) {
        if (this.state === 'pending') {
            this.state = 'fulfilled'
            this.value = value
            this.onFulfilledFns.forEach((fn) => fn(this.value))

        }
    }

    onRejected(err) {
        if (this.state === 'pending') {
            this.state = 'rejected'
            this.err = err;
            this.onRejectedFns.forEach((fn) => fn(this.err))
        }
    }
    then(onresolved, onrejected){
        let myPromiseNext = new myPromise3((resolve, reject) => {
            if (this.state === 'pending' && onresolved) {
                this.onFulfilledFns.push(() => {
                    let x = onresolved(this.value)
                    resolvePromise(myPromiseNext, x, resolve, reject);
                });
            }
            if (this.state === 'pending' && onrejected) {
                this.onRejectedFns.push(() => {
                    let x = onrejected(this.err);
                    resolvePromise(myPromiseNext, x, resolve, reject);
                });
            }

            if (this.state === 'fulfilled') {
                let x = onresolved(this.value)
                resolvePromise(myPromiseNext, x, resolve, reject);
            }
            if (this.state === 'rejected') {
                let x = onrejected(this.err);
                resolvePromise(myPromiseNext, x, resolve, reject);
            }
        })
        return myPromiseNext
    }
}

// 判断是不是promise实例
function isPromise(data) {
    if (data != null 
        && typeof data === 'object' 
        && typeof data.then === 'function') return true;
    return false
}

function resolvePromise(promise2, x, resolve, reject){
    // 循环引用报错
    if(x === promise2){
      // reject报错
      return reject(new TypeError('不能循环引用));
    }
    // 成功或者失败只能调其中一个，不能多次调用
    let called;
    if (isPromise(x)) {
        try {
            x.then.call(x, y => {
                // 成功和失败只能调用一个
                if (called) return;
                called = true;
                // resolve的结果依旧是promise 那就继续解析
                resolvePromise(promise2, y, resolve, reject);
            }, err => {
                if (called) return;
                called = true;
                reject(err);
            })
            
        } catch (e) {
            // 如果调用过了，就不再处理
            if (called) return;
            called = true;
            reject(e); 
        }
    } else {
        resolve(x);
    }
}

let pp = new myPromise((res) => {
        setTimeout(() => res('ok'), 1000)
})
pp.then((res) => {
    return 123
}).then((res) => console.log(res)).then((r) => {
    console.log(r)
})

```

