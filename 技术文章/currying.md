#### 什么叫柯里化？
> 维基百科：把接收多个参数的函数变换成接收一个单一参数（最初函数的第一个参数）的函数，并返回接受剩余的参数而且返回结果的新函数的技术。其由数学家Haskell Brooks Curry提出，并以curry命名。 
> 个人理解：利用闭包，将一个具有多个参数的函数，拆分成多个只有一个参数的函数。


#### 柯里化用途?
1. 参数复用
> 假如一个函数有三个参数，第一个参数比较固定，其余两个参数不固定。可以用柯里化将第一题一个参数的调用当作缓存。
```
function uri(protocol, hostname, pathname){
    return `${protocol}${hostname}${pathname}`
}

uri('https://', 'baidu.com', '/img')
uri('https://', 'jd.com', '/orders')
// 第一个参数每次调用都需要重新写一遍，假如第一个参数很长就恶心了

function uri(prototype){
    return function(hostname, pathname) {
        return `${protocol}${hostname}${pathname}`
    }
}

let httpsUri = uri('https');
httpsUri('baidu.com', '/img')
httpsUri('jd.com', '/orders')
```

2. 提前确认/提前返回
```
// 多浏览器兼容注册事件
const registEvent = (function(){
    if (window.addEventListener) {
        return function(element, eventType, callback, useCapture) {
            element.addEventListener(eventType, function(event) {
                callback.call(element, event)
            }, useCapture);
        }
    } else if (winbdow.attachEvent) {
        return function(element, eventType, callback) {
            element.attachEvent(`on${eventType}`, function(event) {
                callback.call(element, event)
            });
        }
    }
}) ();



registEvent(window, 'click', function(e){
    console.log(this, e)
})
```

3. 延迟执行
// 实现一个函数add，add(1)(2)(3) = 6;add(1, 2, 3)(4) = 10;add(1)(2)(3)(4)(5) = 15;

```
function add(...args){
    const inner = function(...innerArgs) {
        args.push(...innerArgs);
        return inner
    }
    
    // 函数返回会发生隐式转换，从而调用toString。所以在同String里进行加工
    inner.toString = () => {
        return Number(args.reduce((total, current) => total + current))
    }
    return inner
}


function add (...args) {
  const inner = function () {
    if (arguments.length === 0) {
      return args.reduce((total, curr) => total + curr);
    } else {
      args.push(...arguments);
      return inner
    }      
  }
  return inner
}
console.log(add(2)(1, 3, 4)(1)(1)());    // 12
```

#### 实现一个通用的柯里化工具函数

```
function commonCurrying(fn, args = []) {
  let _this = this
  let len = fn.length;
  return function(..._args) {
      args.push(..._args)
      if (args.length < len) {
          return commonCurrying.call(_this, fn, args);
      }
      // 递归边界条件： 收集的参数个数等于原函数的参数个数时， 再将原函数执行。
      return fn.apply(this, args);
  }
}
 
function uri(protocol, hostname, pathname){
    return `${protocol}${hostname}${pathname}`
}


let getUri = commonCurrying(uri);

let baiduUri = getUri('http://')('baidu.com')('/img')

console.log(baiduUri)

const addOne = x => x + 1;
const addTwo = x => x + 2;
const pipe = (input) => (...functions) => functions.reduce(
    (accumulator, currentFunc) => {
        input = currentFunc(accumulator);
        return input;
    },
    input
)


let finalVal = pipe(5)(addOne, addTwo)
console.log(finalVal)