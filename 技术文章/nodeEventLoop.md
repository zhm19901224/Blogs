

#### libuv 
> libuv封装了针对不同操作系统的io的操作，为node提供了异步非阻塞的api，和事件循环

> 因为nodejs是单线程的，所以异步io操作交给了多线程的操作系统区完成


#### Event Loop的6个阶段

1. timers
如果有定时器到期了，会添加进队列。此阶段就是检查定时器到期任务的队列，如果队列为空，就直接进入到下一个阶段。如果队列有任务，则依次执行清空队列。再进入下一阶段。注意：没执行完一个定时器回调函数，就会检查微任务队列，如果有，就先指执行微任务队列，将其清空后再继续执行定时器队列剩余的任务；

2. I/O callbacks
看带有io操作的api有没有获取到结果的，比如fs.readFile，文件系统api会调用操作系统内核的方法处理（js单线程、系统是多核多线程）, 如果处理完了，会将结果告诉libuv，这时会将回调函数压入I/O taskQueue。
本阶段就是检查taskQueue，如果有就执行，没有就进入下个阶段

3. idle prepare。 等待阶段。不做研究

4. poll 轮训
检查处理定时器taskQueue。
4.1 如果队列中有setImmediat，就终止poll阶段，去check阶段执行setImmediat任务队列。
4.2 如果队列里没有setImmediat就查看有没有定时器setTimeout/setInterval到期，有的话就跳过check阶段，直接去timers阶段执行定时器到期任务队列

5. check

执行setImmediat。只要执行完就立刻到timers阶段了。

6. closeCallback 忽略不做研究



process.nextTick、promise.then在任意两个阶段中间，只要有未执行的，就执行nextTick回代调、promise.then回调。nextTick优先于Promise.then


从事件循环一开始就去检查微任务，然后执行当前阶段。在当前阶段和下个阶段之间，又会去优先检查微任务


#### 宏任务、微任务

微任务：Process.nexttick、promise.then

宏任务：setInterval、setTimeout、setImmidiat。。。

关于微任务，必须要知道三点：
1. 微任务事件循环开始执行前会执行，直到将微任务队列消费干净，开始进入事件循环。
2. 在微任务队列中，通常promise.nexTick优先级高于Promise.then。**但是有一种情况特殊, promise.then回调中，promise.then优先级高于process.nextTick**
```
Promise.resolve().then(() => {
    process.nextTick(() => {
        console.log(2)
    })
    
    Promise.resolve().then(() => {
        console.log(1)
    })
})
// 1 2
```
> 除此情况之外，异步队列如果有Promise.then和process.nextTick, 一定是process.nextTick优先执行
3. 在每个事件循环阶段结束，下个阶段到来之前，都会去检查微任务度队列，如果有微任务，就依次执行;timer阶段、check阶段，每完成一个队列中的任务，就会去检查微任务队列；

#### 面对复杂问题，我的的思考方式

1. 想想当前处于在事件循环什么阶段？
2. 当前的阶段对应的宏任务队列有没有任务需要被执行？
3. 微任务队列里有没有任务？阶段结束后需要执行哪些微任务？
4. 阶段结束之后，各个阶段对应的任务队列发生了什么变化？

举个例子
```
setTimeout(() => {
    process.nextTick(() => {
        console.log(3)
    })
}, 0)

Promise.resolve().then(() => {
    process.nextTick(() => {
        console.log(2)
    })
    
    Promise.resolve().then(() => {
        console.log(1)
    })
})

/* 
step 1 当上述代码运行过后，各个队列情况
microTaskQueue 
[
    () => {
        process.nextTick(() => {
            console.log(2)
        })
        
        Promise.resolve().then(() => {
            console.log(1)
        })
    }
]

由于定时器时间为0，相当于1ms，定时器到期，所以进入会进入定时器队列

timersTaskQueue [
    () => {
        process.nextTick(() => {
            console.log(3)
        })
    }
]



step 2. 时间循环运行之前，先将微任务队列执行清空
() => {
    process.nextTick(() => {
        console.log(2)
    })

    Promise.resolve().then(() => {
        console.log(1)
    })
}
执行过后，微任务队列变成[promise.then, nextTick]

step 3. 开始事件循环，进入timers阶段，检查timersTaskQueue，发现有任务
() => {
    process.nextTick(() => {
        console.log(3)
    })
}
将任务执行完成后，微任务队列发生了变化，队列末尾多了一项。由于开始时在then内部情况很特殊，所以promise.then的优先级优先于nextTick
[
    Promise.resolve().then(() => {
        console.log(1)
    }),
    process.nextTick(() => {
        console.log(2)
    }),
    process.nextTick(() => {
        console.log(2)
    })
]

step 4. 在进入io callback阶段之前，检查执行微任务队列。
打印 1 2 3


step 5. 事件循环依次进行，发现各个阶段宏任务、微任务队列都没有任务了，停止事件循环
*/

```

#### setTimeout(0), setImmediate谁先执行？

案例1
```
setTimeout(() => {
    console.log(1)
}, 0)


setImmediate(() => {
    console.log(2)
})

// 2 1  setImmediate先执行
```

案例2
```

setTimeout(() => {
    console.log(1)
}, 0)


setImmediate(() => {
    console.log(2)
})

console.log(123)
// 123 1 2 setTimeout先执行
```

> 案例1案例2，为何执行顺序不一样？！我们先从事件循环阶段分析。setTimeout(()=>{},0),相当于setTimeout(()=>{},1)，是有至少1ms延迟的。

> 对于案例2，在进入事件循环处理异步任务之前，先执行同步任务，并且定时器开始计时。假如同步任务完成之前，定时器到期了，那么在事件循环之前，timers对应的taskQueue内就有可以执行的回调了。

> 此时对于在第一个阶段Timers,直接将timeout内部的回调执行了，接下来到后面运行到check阶段才将setImmediate给执行


> 而对于案例1，没有同步代码，事件循环开始，定时器没到期，timer阶段的任务队列是空的。那就先往下执行。执行到poll阶段发现队列中有setImmediate，就直接去check阶段处理setImmediate的任务了，最后等定时器到期了，并且停留在timers阶段才将timeout任务执行。

> 再来看看案例3

案例3
```
require('fs').readFile('a.txt', 'utf-8', () => {
    console.log(123)
    setImmediate(() => {
        console.log(1)
    })

    setTimeout(() => {
        console.log(2)
    }, 0)
})
// 123 1 2
```

> 思考，为何有同步代码等定时器到期，却还是setImmediate比setTimeout先执行？

> 看过前面“面对复杂问题，我的的思考方式”的同学应该知道。要先看当前处于什么阶段。在案例3中，已经处于io callback阶段了。接下来是poll、check。所以一定是setImmediate先在本次循环的check阶段执行。至于setTimeout，有可能在下次循环的timers阶段执行，如果定时器时间长一点，甚至可能好几次事件循环过去了，timers的任务队列都是空的，直到定时器到期。

#### 终极面试题

```
const { readFile } = require('fs')

setImmediate(() => console.log(11))
setImmediate(() => console.log(12))
setImmediate(() => console.log(13))

Promise.resolve().then(() => {
    console.log(5)
    setImmediate(() => console.log(14))
})

readFile('a.txt', 'utf-8', data => {
    console.log(15)
    readFile('b.txt', 'utf-8', data => {
        console.log(27)
        setImmediate(() => console.log(28))
    })

    setImmediate(() => {
        console.log(17)
        Promise.resolve().then(() => {
            console.log(18)
            process.nextTick(() => console.log(20))
        })
        .then(() => {
            console.log(19)
        })
    })

    setImmediate(() => {        
        console.log(21)

        process.nextTick(() => console.log(23))
        console.log(22)
        process.nextTick(() => console.log(24))

        readFile('a.txt', 'utf-8', data => {
            console.log(29)
            setImmediate(() => console.log(30))
            setTimeout(() => console.log(31), 0)
        })

    })

    process.nextTick(() => {
        console.log(16)
    })

    setTimeout(() => console.log(25), 0)

    setTimeout(() => console.log(26), 0)

})

setTimeout(() => console.log(6), 0)  

setTimeout(() => {
    console.log(7)
    process.nextTick(() => {
        console.log(8)
    })
}, 0)

setTimeout(() => console.log(9), 0)
setTimeout(() => console.log(10), 0)

process.nextTick(() => console.log(1))
process.nextTick(() => {
    console.log(2)
    process.nextTick(() => console.log(4))
})

process.nextTick(() => console.log(3))


```

这份代码执行结果是1 - 31顺序打印的。按照我的思考方式，去搞清楚为何是按照这个执行顺序执行。

要点：想当前哪个阶段、不断去记录各个阶段的队列。

#### 最后

码字不易，转载请注明出处