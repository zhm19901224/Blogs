### CO库的基本原理

##### 前言
最近在回顾es6的很多内容，在看到Generator的时候，想起了几年前工作中书写node代码用Generator生成器函数、Promise、co处理callback hell的场景。当时只觉得co很好用，能够以同步的书写方式处理promise的结果，但并没有很深入琢磨内部原理。昨天看了一下co源码，基本了理解了其内部实现。实在巧妙，所以总结一下。

##### Generator基本用法
```
function *gen(){
    yield 1;
    yield 2
    yield 3
}
let runner = gen();
runner.next();  // { done: false, value: 1}
runner.next();  // { done: false, value: 2}
runner.next();  // { done: true, value: 3}
runner.next();  // { done: true, value: undefined}
```

##### next的参数问题
next的唯一参数会成为generator函数上一个yield表达式的返回值
```
function *gen(){
    let res = yield 1
    console.log(res)
    yield 2
}
let runner = gen();
console.log(runner.next())
console.log(runner.next('这是res，第一个yield表达式的返回值'))
/**
打印结果入下
{ value: 1, done: false }
这是res，第一个yield表达式的返回值
{ value: 2, done: false }
*/
```


##### co库基本用法
```
function request(params, time){
    return new Promise((resolve) => {
        setTimeout(() => {
            resolve({
                code: 0,
                message: 'success',
                data: params
            })
        }, time)
    });
}

co(function* (){
    const res = yield request('第一个请求结果', 2000)
    console.log('res', res)
    /* 结果为异步返回的value
    {
        code: 0,
        message: 'success',
        data: params
    }
    */

    const res2 = yield request('第二个请求结果', 2000)
    console.log('res2', res2)
})

```
> co实现原理推断

正如上面代码所示，res可以以同步方式拿到request()这个Promise对象then中的结果值。结合next函数参数的原理，可以推断，处理第一个yield，一定是执行了两次next方法。一次调用next，拿到generator返回的Promise，并在then中提取到fulfilled状态的value值，将这个值再次调用next(value)。 这时，拿到了第二个res2 Prmise，并且第一个res已经赋上value值了。不管要接受有多少个yield，都要不断循环或者递归，直到done为true结束。

> 根据co源码，将核心原理压缩成以下几行代码

```
function request(params, time){
    return new Promise((resolve) => {
        setTimeout(() => {
            resolve({
                code: 0,
                message: 'success',
                data: params
            })
        }, time)
    });
}

function co(generatorFn){
    let runner = generatorFn();
    function getAndRunAllPromise(...args){
        // ...ars第一次执行没有值
        let current = runner.next(...args)
        if (!current.done) {
            // 开始递归，当promise是fulfilled状态时，下次递归函数的参数就会是onResolved回调函数中的成功值
            return current.value.then(getAndRunAllPromise)
        }
        return
    }
    getAndRunAllPromise();
}


co(function*(){
    let res1 = yield request(111, 1000);
    console.log(res1)
    let res2 = yield request(222, 1000);
    console.log(res2)

});


```

> 可以看书递归函数也可以通过promise.then延迟执行，从第二次运行getAndRunAllPromise时开始，其实这个函数内部做了三件事，1.拿到当次yield的promise对象。2.通过next，将上一次执行该函数是，promise的完成结果传入到上一次yield表达式中;3.等待完成这一次的Promise pennding状态过度到fulfilled态，并将结果，传入下一次递归中


> co源码其实大部分都是在做类型判断，将不是promise的yield，转化成为promise。es8推出的async、await与co极其相似！每次await后面接的内容，都会被转化为Promise。我猜测，async function大概率参考了co的执行原理吧！