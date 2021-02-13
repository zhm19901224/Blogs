### 设计原则

<UNIX/LINUX设计哲学>
1. 小即是美
    比如koa，express就太重了
2. 让每个程序只做好一件事
3. 快速建立原型（先满足基本需求）
    快速建立一个满足用户基本需求的产品，再根据用户的反馈迭代
4. 可以移植性，比高效更重要
5. 采用纯文本存储数据, 可读性更高
6. 可复用性
7. 每个程序都是过滤器(类似中间件，面向切面aop)
8. 允许用户定制环境
9. 使用小写字母，尽量简短
10. 满足90%用户需求就可以了


### SOLID五大设计原则
1. single 单一职责原则
2. open-close 开放封闭原则，对扩展开放，对修改封闭
3. Li 李氏置换原则 父类能用的地方，子类就能用，子类可以覆盖父类
4. interface 接口独立原则，和单一职责原则一样，避免“胖接口”
5. dependence 依赖倒置换原则，依赖于抽象接口，不依赖具体实现


> js中后三个体现比较少，因为不是纯面向对象

### 设计模式三大种类概览
1. 创建型
    1.1 工厂模式
     简单工厂、抽象工厂、建造者
    1.2 单例模式
    1.3 建造者（js不常用）
2. 组合型（结构型）
    2.1 适配器
    2.2 装饰器
    2.3 代理模式
    2.4 外观模式
    2.5 桥接、组合、享元（js不常用）
3. 行为型
    3.1 观察者
    3.2 迭代器
    3.3 职责链
    3.4 策略模式
    3.5 状态模式
    3.6 模版方法模式、命令模式 （js不常用）
    3.7 备忘录、访问者、中介者、解释器(js不常用)

### 如何学习设计模式
1. 必须清楚每个设计模式的设计用意、道理；
2. 写一些真正的实战场景；


### 创建型
#### 1. 简单工厂（静态工厂模式）
> 定义一个类(js不一定，可以是一个函数) 来负责创建其他**类的实例**，被创建的实例通常都具有**共同的父类**。

Demo1
```
class Car {
    constructor(carName, carNumber){
        this.carName = carName
        this.carNumber = carNumber
    }
}
// 快车
class FastCar extends Car {
    constructor(carName, carNumber, price){
        super(carName, carNumber)
        this.price = 2;
    }
}
// 专车
class ProCar extends Car {
    constructor(carName, carNumber, price){
        super(carName, carNumber)
        this.price = 5;
    }
}
// 专车
function getCar(type, carName, carNumber) {
    if (type === 'fast') {
        return new FastCar(carName, carNumber)
    } else if (type === 'pro'){
        return new ProCar(carName, carNumber)
    } else {
        throw new Error('type not exists')
    }
}

const fastCar1 = getCar('fast', '奥迪A6', '京88888')

console.log(fastCar1)
```

> 以上例子，只需知道要定义车类型、名称、车牌号，就能生成实例，使用者无需知道具体生成过程
> 如果多处使用了快车实力，到处new，一旦类名发生改变，那将是灾难。
> 不直接用new 生产实例，对于链式调用很友好

Demo2
```
window.$ = function(selector){
    return new JQuery(selector)
}

React.createElement // 创建Vnode实例
```

#### 2. 抽象工厂
> 假如有如下场景，一个餐馆，生产具体的饭菜，有菜、有汤。红烧肉、罗宋汤都是具体的产品类。但是又属于两个不同的产品类别。菜能吃、汤能喝。而且生产的餐馆应该有两个不同的流水线，一个专门做菜、一个专门负责做汤。这个业务场景需要抽象工厂

> 4个重要角色
1. 抽象工厂类：定义工厂类的结构，抽象类不能被实例化，只能用于继承，属性重写
2. 工厂类：根据抽象工厂类继承创建，内部产生不同类别的产品类（生产菜、生产汤）
3. 抽象产品类：具体产品的某个类别（比如菜类、汤类），定义其类别特有的属性（吃、喝）
4. 具体产品类：不同的具体产品继承自不同的抽象产品类(红烧肉继承菜抽象类、罗宋汤继承汤抽象类)

```
class AbstractRestaurant {
    constructor (){
      if (new.target === AbstractRestaurant) {
        throw new Error();
      }
    }
  
  
    createDish(type: any){
      throw new Error('抽象类不能实例化')
    }
  
  
    createSoup(type: any){
      throw new Error('抽象类不能实例化')
    }
  }
  
  
  class Restaurant extends AbstractRestaurant {
    constructor(){
      super();
    }
  
  
    createDish(type: any){
      switch(type){
        case '宫保鸡丁': 
          return new Gongbaojiding();
          break
      }
    }
  }
  
  
  class AbsoluteDish {
    kind: string;
    constructor(){
      if(new.target === AbsoluteDish){
        throw new Error('抽象类不能实例化')
      }
      this.kind = '菜'
    }
  }
  
  
  
  class Gongbaojiding extends AbsoluteDish {
    type: string;
    constructor(){
      super();
      this.type = '宫保鸡丁';
    }
  
  
    eat(){
      console.log(`${this.kind} —— ${this.type}真香！`)
    }
  }
  
  
  const restaurantfactory = new Restaurant();
  (restaurantfactory.createDish('宫保鸡丁') as any).eat();
```


#### 3. 单例

> 只能实例化一次，不论多少个new，得到了都是同一个实例
> 应用：路由、vue、redux的store

```
class SingleInstance {
    constructor(){
        // 。。。类的初始化
        if (SingleInstance._instance) return SingleInstance._instance;
        SingleInstance._instance = this;
    }

}

SingleInstance.getInstance = (...args) => {
    if (SingleInstance._instance) return SingleInstance._instance
    SingleInstance._instance = new SingleInstance(...args);
    return SingleInstance._instance
}

```


### 组合型

#### 1. 适配器
> 一个适配器函数，内部调用久接口不改变久接口，返回新的返回值，vue的computed就是标准的适配器
> so easy

#### 2. 装饰器
> 本质一个函数，用于动态给一个类添加属性，或者给一个类的属性添加属性描述符(desciption)
> es7 decorator babel已经有插件可以用
```
// https://babeljs.io/docs/en/babel-plugin-proposal-decorators
// npm i @babel/plugin-proposal-decorators -D
// .babelrc 中
//  "plugins": [["@babel/plugin-proposal-decorators", { "legacy": true }]]

// 动态给Animal类添加introduce方法
function DecoIntro(target){
    target.prototype.introduce = function () {
        console.log(`hi, my name is ${this.name}`)
    }
}

@DecoIntro
class Animal {
    constructor(name){ this.name = name }
}

```

> 如果需要给装饰器函数传递参数
```
function DecoIntro(area){
    return function(target){
        target.prototype.introduce = function () {
            console.log(`hi, my name is ${this.name},i am from ${area}`)
        }
    }

}

@DecoIntro('China')
class Animal {
    constructor(name){ this.name = name }
}
```

> **实现mixin**，将**多个对象内的方法**混合到**类的原型方法**中去;
```

function mixin(...list){
    return function(target){
        Object.assign(target.prototype, ...list)
    }

}

let ob1 = {
    eat() {
        console.log('i am eating')
    }
}

let ob2 = {
    sleep() {
        console.log('zzz....')
    }
}

@mixin(ob1, ob2)
class Animal {
    constructor(name){ this.name = name }
}

let a = new Animal('tom')
console.log(a.sleep())
```

> React高阶组件hoc的高阶组件其实就是装饰器，一个纯函数，参数是组件类。 下面用es7 decorator改写

原先写法
```
// content.jsx
import React, { Component } from 'react'
import withHeadFoot from './withHeadFoot'



class Content extends Component {
    render(){
        return <section>这是内容主体</section>
    }
}
export default withHeadFoot(Content)


// withHeadBody.jsx
import React, { Component } from 'react'

export default (Content) => {
    return class withHeadFoot extends Component {
        render(){
            return (
                <>
                    <header>这是头部</header>
                    <Content {...this.props} />
                    <footer>这是尾部</footer>
                </>
            )
        }
    
    }
}

```

如今写法, 优雅很多
```
import React, { Component } from 'react'
import withHeadFoot from './withHeadFoot'
@withHeadFoot
class Content extends Component {
    render(){
        return <section>这是内容主体</section>
    }
}

export default Content
```

#### 装饰类的属性
> 装饰器函数有三个参数target、key、{ value, writable, configurable, enumerable }
@babel/plugin-proposal-class-properties
```
import React, { Component } from 'react'
import withHeadFoot from './withHeadFoot'

function readonly(target, key, description){
    description.writable = false;
    return description
}

class Content extends Component {

    @readonly
    getList(){

    }

    render(){
        return <section>这是内容主体</section>
    }
}

export default Content
```

#### 代理模式
> 两个角色：代理对象Proxy、目标对象Target

1. 拦截器
axios.interceptor   vueRouter的beforeEach
响应式框架，通过Object.defindeProperty、Proxy拦截对数据的getter和setter操作


> 利用Object.defineProperties模拟一个数据代理器拦截器
```
let data = {
    aaa: 123,
    bbb: 456
}
function proxy(data = {}){
    let keys = Object.keys(data)
    let proxyData = {}
    let getSetConfig = {}
    keys.forEach(key => {
        let newKey = '_'+key
        proxyData[newKey] = data[key];
        getSetConfig[key] = {
            get(){
                console.log('要设置的key是'+key)
                return proxyData[newKey]
            },
            set(value){
                console.log('set拦截到了，value是'+value)
                this[newKey] = value
            }
        }
    })
    Object.defineProperties(proxyData, getSetConfig);
    return proxyData
}
let pData = proxy(data)
console.log(pData.aaa = 'zhm', pData)
```

> Object.defineProperties极其的恶心，它的get和set一旦写不好就成了循环调用，最后调用栈溢出。。。。所以只能通过利用闭包，每次get、set的其实都不是真实getset的属性。这玩意这么反人类，注定要退出历史，被Proxy取代



> 来看看es6 Proxy多优雅, 不再用克隆，不再用闭包
```
let data = {
    aaa: 123,
    bbb: 456,
    ccc: [123, 456]
}

let pData = new Proxy(data, {
    get(target, key){
        console.log('get到了')
        return target[key]
    },
    set(target, key, value){
        console.log('set了')
        target[key] = value
    }
});

pData.ddd = 888         // 新增属性也能被拦截到！
console.log(pData)
```

2. 缓存代理
vue computedStyle的实现，每次访问计算属性，先去看相关的原属性有无改变，没改变久返回返回缓存，改变了，就拼装新计算属性，缓存后再返回
3. 保护代理
nginx网管，收到大量请求时，过滤垃圾请求、无效请求
4. 正向代理
源服务器对客户端不可见
5. 反响代理
客户端对源服务器不可见，只能看到代理服务器

##### 思考：如何用装饰器模式、代理模式给React添加一个计算属性的getter、setter

#### 外观模式
> 将下层接口统一封装进一个上层接口，供使用者使用。不符合单一职责原则和开放封闭原则，谨慎使用

1. 在开发node模块的时候，很多时候会把子模块聚合在一个index.js内部统一导出，为使用者提供了很大方便，但是tree-shaking可能会有问题了
2. 开发前端项目，对请求进行二次封装，通常会将所有请求的promise方法收拢在一个api模块内，统一调用，也方便进行测试


### 行为型

#### 1. 观察者模式
##### 观察者模式
> 两个角色：观察者类、主题类

> 一个符合UML类图的标准观察者
```

class Subject {
    constructor(){
        this.state = false;
        this.observers = []
    }
    getState() {
        return this.state
    }
    setState(state) {
        if (state) this.notifyAll()
        this.state = state;
    }
    attach(ob){
        this.observers.push(ob)
    }
    notifyAll(){
        // 触发所有观察者的事件
        this.observers.forEach((ob,index) => ob.update(index+'触发了'))
    }
}


class Observer {
    constructor(name, subject){
        this.name = name;
        this.subject = subject; // 主题类用于将当前观察者注册到主题中
        subject.attach(this)
    }

    update(words){
        // 要触发的事件
        console.log(words)
    }
}

let sub = new Subject()

let ob1 = new Observer('ob1', sub)
let ob2 = new Observer('ob2', sub)

sub.setState(true)
```

> 上面的观察者类，自己就将自己注册到主题类中。下面实现一个简单的event事件模型
```

class myEvent {
    constructor() {
        this.sub = {};  // 保存所有事件的队列
    }

    on(eventName, fn){
        let sub = this.sub
        if (sub[eventName]) {
            sub[eventName].push(fn)
        } else {
            sub[eventName] = [fn]
        }
    }
    off(eventName){
        if (this.sub[eventName]) this.sub[eventName] = null
    }
    emit(eventName){
        if (this.sub[eventName]) this.sub[eventName].forEach(fn => fn())
    }
}

let e1 = new myEvent()
e1.on('call', () => console.log('haha'))
e1.emit('call')
```

> 常见的应用: redux、Event对象、promise.then对异步场景的处理。下面是一个简单的redux

```

const redux = (function() {
    let state = {}
    let subs = []
    function createStore(reducer, defaultState){
        if (defaultState) Object.assign(state, defaultState)
        return {
            getState(){
                return state
            },
            subscribe(fn){
                subs.push(fn)
            },
            dispatch(action){
                state = reducer(state, action)
                subs.forEach(fn => fn())
            }
        }
    }
    return {
        createStore
    }
})()

function test() {
    const { createStore, subscribe, dispatch } = redux
    let defaultState = { num: 1 }
    let reducer = (state = defaultState, action) => {
        switch (action.type) {
            case 'add':
                let nState = {...state}
                nState.num += 1;
                return nState
            default: 
                return state
        }
    }
    let action = { type: 'add' }
    let myEvent = () => console.log('检测到store的变化了')
    // 开始测试
    let store = createStore(reducer, defaultState)
    store.subscribe(myEvent)
    store.dispatch(action)
    console.log(store.getState())
}

test();

```

> 可以看出来redux核心代码也就是20多行，但是测试的demo比源码核心还要多。。。。。。。。。。



#### 迭代器模式

> 使一个线性数据结构的数据，能够可以通过接口来顺序访问

> 当你身处es3的年代，js没有forEach的时候？？？
```
Array.prototype.myForEach = function(fn){
    if (Object.prototype.toString.call(this) !== '[object Array]') throw new Error('array is required!')
    var i = 0;
    var arr = this
    while(i < arr.length){
        fn(arr[i], i, arr);
        i++
    }
}


let arr = [111, 222, 333]
arr.myForEach((item, i) => {
    console.log(item, i)
})

```

> 如今es6有了iterator概念，可以给任意数据结构添加可迭代接口，统一用for of语法进行访问非常方便

#### 状态模式

> 前端很多场景都用到状态，状态模式两个角色是主体类、上下文类。
> 上下文类用于操作状态，如get、set。主体类比如一个组件，如果想改变状态，需要传入上下文。这样比较复杂。

> 应用场景: react等框架state、promise三状态、http status码等

> 个人理解，对于需要状态的业务场景，一个类自身提供getState、setState这样的接口就可以了，适当的降低复杂度总是好的！


#### 策略模式
> 将多个if else的复杂业务逻辑用不同的配置对象或者creator代替，凡是遇到配置文件的地方大多使用了策略模式。