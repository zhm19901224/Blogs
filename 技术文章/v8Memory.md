#### 前言
node开发和前端js开发不太一样。因为运行在服务器上，而服务器的内存资源可谓寸土寸金，所以对程序的要求要更加严谨；

#### V8的内存管理
node本质是一个由V8引擎操控的C++程序，可以理解为运行js的一种虚拟机。在node进程运行后，开辟内存并对内存进行三种分段：

1. 代码：实际运行的代码，从硬盘中读取到内存中；
2. 栈内存：存储值类型数据、引用类型数据指针、代码程序控制流的指针；
3. 堆内存：用于引用类型数据、闭包；


#### nodejs程序中查看内存
```
console.log(process.memoryUsage())
/*
{
  rss: 22790144,
  heapTotal: 4251648,
  heapUsed: 2138336,
  external: 680547
}
*.
```

1. rss：node进程分配到的常驻内存。包含C++对象、js对象、js代码；
2. heapTotal：v8的堆内内存总值;
3. heapUsed：v8使用了的堆内内存;
4. external：V8管理的内存中，有多少C++对象绑定到了js对象上，这部分C++对象的内存;

> 上面看着有点儿绕的，简单的说node内存 = 堆内内存 + 堆外内存; 
> Buffer属于堆外内存，不受v8管理，由操作系统直接管理；
> 堆内内存，我们就关心heapUsed、heapTotal就好了

> node进程的内存使用上限，64位系统约1.4G，32位系统约0.7G。这个是上限，一开始堆内存不会分配很多，当发现堆内存不够用时才会继续申请，直到这个上限

> 突破限制，在启动node时，加max-old-space-size=2048或max-new-space-size=2048来提升内存分配上限。一般用不上，如果内存激增说明内存泄露严重。

#### v8垃圾回收机制(garbage collection)
1. 分代式垃圾回收机制
> nodejs将内存划分为新声代内存(存在事件短)、老生代内存(存在时间长，如全局变量)。
> 堆内存 = 新生代内存 + 老生代内存

新生代内存采用scavenge算法进行垃圾回收；

2. 新生代scavenge算法
基于cheney回收算法：
    step1. 将内存划分为使用中的from空间，空闲的to空间
    step2. 在from空间内检查还要继续引用的存活对象;
    step3. 对这些存活对象做迁移判断，如果这些对象，被scavenge算法回收过，说明能活挺长时间，那么将这些对象迁移到老生代内存中；如果to空间剩余空间不超过堆内存25%了，那么也将这些存活对象放到老生代内存中（不这么做，to、from翻转后，from的可用内存不足25%，而to变成75%就太可怕了）。剩下的被复制到to空间中，还当作新生代。（复制少量还好，复制大量对象就有效率问题了）
    step4. 将from空间内剩余要释放的对象释放掉，回收内存。
    step5. 将from、to空间互换(to空间相当于临时区)
缺点： 以空间换时间，浪费了堆内存的一部分（to空间）

3. 老生代Mark-Sweep(标记、清除) & Mark-Compact(标记、整理)算法
> 对于老生代内存，存活的对象很多，如果采用scavenge算法，那么step3，从from空间复制到to空间的复制操作效率很低，而且scavenge浪费了一半的内存空间不友好；所以采用了标记、清除算法;

> 标记清除算法很简单，标记可以存活的对象，删除没被标记的对象。剩下的就是释放掉的内存空间,下面用数组来模拟这个过程。假设开辟的老生代内存是长度为5的数组，偶数位置的b、d是可以存活的对象。marled是存活标记：

step1. 标记: [a(marked), b, c(marked), d, e(marked)]
step2. 清除: [a(marked), , c(marked), , e(marked)]

> 这时候存在一个问题，step3存在大量的内存碎片占着数组位置，所以就有了Mark-Compact标记整理，每次清除完，都将后面的存活对象移动到前面存活对象的位置。

step1. 标记: [a(marked), b, c(marked), d, e(marked)]
step2. 清除: [a(marked), , c(marked), d, e(marked)]
step3. 整理: [a(marked), c(marked), d, e(marked), empty]
step4. 清除: [a(marked), c(marked), , e(marked), empty]
step5. 整理: [a(marked), c(marked), e(marked), empty, empty]
step6. 整理完成: [a, c, e]

> 增量标记、增量整理
上面的执行过程是非常繁琐的，而js又是单线程的，每次执行垃圾回收都要阻塞线程，让js代码停止执行。所以上面标记、整理过程变成了增量式的，每执行完一个步骤，就让js代码执行一会。这样回收时间变成原先的1/6，效率有明显提升


#### 内存泄露的几种情况

1. 闭包
> 外部作用域变量outerVar引用了内部作用域的变量innerVar，内层函数执行完出栈了，其内层作用域的变量还被外层作用域的outerVar变量引用，不被回收。


> node模块导出的变量副本，这就是一种内存泄露。它用闭包保证了各个模块之间的隔离，不会将变量挂载到global上，但是闭包本身不被释放，第一次导出后，执行结果会被成为内存中的缓存。直到node进程结束，才会被释放。
```
(function(exports, require, module, __filename, __dirname) {
    const data = 123;
    module.exports = data
});
```

2. 全局变量
> 函数作用域内的局部变量，只要函数执行完，弹出执行环境栈，就被回收。

> 而全局变量只能等node进程停止，才能被释放。它存在于老生代中，常驻内存的对象。尽量避免在全局创建变量，如果创建了，在用完时，可以赋值null，或者delete。

3. 内存缓存
> RJS模块就利用了内存缓存，执行并加载过就存入缓存，下次直接从缓存中去严格上说属于内存泄露。

> 看过我分片上传demo源码的朋友会发现。有一处很明显的内存泄露，我用一个js对象放在模块顶部，用于记录文件上传的具体状态，这个变量不在某个函数作用域内，不可能被回收。除非进程中断了，中断后再启动，也起不到持久化数据作用了。所以这种写法是有问题的。生成不可能这么用


4. 有创建特性的api
> 对于定时器timer相关函数的返回值，会创建一个定时器信息的对象，这个对象无法自动被垃圾回收，必须调用清除定时器相关函数。类似的还有map、filter等函数，都会创建一个新的数组，如果不被变量引用，也会造成内存泄露。

5. 对象间的循环引用
```
var first = {
    a: last
}

var last = {
    b: first
}
```
> 在一此垃圾回收过程中，两个变量始终是被引用状态，不会被标记

6. 循环创建stream可读流


#### 内存泄露的排查与监控
1. chrome devtools中使用了inspect调试后，在控制台Performance板块中点击录制按钮。在timeline的图表中浅蓝色区域显示不同时间节点，内存占用情况；

2. node --trace_gc > gc.log server.js  可以在gc.log文件夹中看到几个垃圾回收周期中，新生代、老生代的不同垃圾回收算法的执行时间、回收内存的大小

3. node-heapdump
> 借助node-headdump，抓拍内存快照。在chrome devtools中进行分析。
step1. 安装npm install heapdump
step2. 代码首行require('heapdumo'), 启动服务
step3. ps ax | grep <入口>.js  找到node进程的pid
step4. kill -USR2 <pid> 终止进程
step5. 打开chrome devtools的Memory，点击底部的load，选择step4生成的.json快照文件
step6. 查看快照，分析内存变化情况



#### 参考

田勇强《深入浅出nodejs》

nodejs中文网 http://nodejs.cn/api/process.html#process_process_memoryusage

转载本文，请注明出处，多谢！
