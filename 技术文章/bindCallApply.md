## 理解Js中bind、call、apply的最好方式就是重写一遍
#### 铺垫： this，Js中的this指的是当前函数的调用者，通常是对象。this的指代通常有七种情形：
1. 在全局作用域内，this指代window或者global(node);
2. 非严格模式下，在全局作用域的函数内部，this指代window或者global(node);
3. 在严格模式下，在全局作用域的函数内部，this指代undefined；
4. 在构造函数中，this指代要实例化的新对象；
5. 在对象的某个对象方法内部，this指代该方法；
6. 在es2015箭头函数内部，本身没有this一说，箭头函数会借用其声明时所在静态作用域的this。例如：
```
var name = 'abc';
let ob = {
    name: 'def',
    say: function () {
        return () => {
            console.log(this.name);
        }
    }
}

ob.say()();     
// 箭头函数声明在say fn这个函数作用域内，say fn内部的this时ob，所以this是指代ob
```
7. 匿名函数的this都是undefined；常见于各种callback；
8. dom绑定事件，this指代绑定的目标对象；相当于第5种；
```
window.onclick = function() {
    console.log(this)
}
```

#### Function.prototype.bind: 改变调用当前函数的上下文对象，使函数执行时，内部的this变成绑定的上下文对象。其返回值是原函数的副本(只不过this变了，或者参数发生变化);
例如以下例子：
```
let obA = {
    a: 'a',
    test: function(...params) {
        console.log(params, this.a)
    }
};

let obB = {
    a: 'bbb'
}

obA.test(2);        // [2] a


let copy = obA.test.bind(obB, 111, 222);

copy(333);  // [111, 222, 333]  bbb
```
> 用一个生活中的例子形容bind，有个小孩去打酱油作为函数执行体，入参是钱，返回是酱油。但是爸爸派他去和妈妈派他去不一样。this就是指谁派过去的。

> 重写bind分四个步骤：
1. 先拿到原函数的this上线文对象；
2. 获取到bind函数的第一个参数(要替换的上线文对象Context)、其余参数(执行时传入的参数)
3. 以闭包函数的方式制作一个原函数的副本，在闭包函数中，需要处理bind合并参数，并将this上线文替换成Context对象。
4. 将副本返回；
```
    Function.prototype.myBind = function(){
        // fn为要调用的目标函数函数体
        let fn = this;
        // 调用该函数的上下文对象, bind的第一个参数
        let context = [...arguments].shift();
        // 函数调用的参数， bind的第二到最后一个参数
        let params = [...arguments].slice(1);
        return function () {
            // 此闭包函数为真正要执行时的调用环境，参数为之前bind绑定时传入的              参数，和真实调用时又传入的参数。所以需要合并参数。
            let callingArgs = params.concat([...arguments]);
            context.tempFn = fn;
            // 此时原函数内部this发生了变化，this指向了context对象了，需要            将执行结果通过闭包形式输出
            let bindFnResult = context.tempFn(...callingArgs);
            delete context.tempFn;
            return bindFnResult;
        }
    }
```


#### Function.protype.call 将原函数的调用对象替换，并执行函数，第一个参数是替换的context对象，其余参数是函数执行参数。
> 与bind相同点是, 都改变了函数调用者的指向。
> 与bind区别是，bind返回是原函数副本，call、apply是直接执行原函数。

> 重写call
```
Function.prototype.myCall = function(){
    let fn = this;
    let [context, ...paramsArr] = [...arguments];
    context.tempFn = fn;
    let res = context.tempFn(...paramsArr);
    delete context.tempFn;
    return res;
}
```


#### Function.protype.apply 和 call一摸一样，唯一区别是第二个参数是数组，函数执行时会将参数数组打散成若干个参数。
> 重写apply
```
Function.prototype.myApply= function(){
    // 这里忽略参数类型校验
    let fn = this;
    let [context, paramsArr] = [...arguments];
    context.tempFn =  fn;
    let res = context.tempFn(...paramsArr);
    delete context.tempFn;
    return res;
}
```


### 总结
至此，this、bind、call、apply原理应该掌握得绝对清晰，足以应对开发过程中的任何问题，大家可以随意转载，但请大佬给个star吧！不要下次一定，哈哈！