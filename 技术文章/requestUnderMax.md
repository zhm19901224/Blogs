#### 为啥搞这个?

最近在用原声nodejs写一个大文件分片上传、断点续传的案例。在完成到90%的时候，发现了一个很头疼的问题。对于分片大小，网上的建议是2-5M之间，既保证灵活性，又不至于分片过多，在网络请求上给服务器添太大压力。但对于一个5-6Gb的视频文件(绿色的)来说，以5M的分片大小，还是要切割上千片。服务器压力先不考虑，光浏览器一下子建立上千个连接池都很够呛啊！我写的Promise.All，本来的逻辑是所有分片都上传成功了，发送一个merge请求到后端，未曾想浏览器让上千个请求直接给搞崩了。

所以，只能通过控制单位并发量来解决这个问题。chrome的同一域名下，可以建立的tcp连接数是6。也就是同一时刻我们的并发请求不能超过6，超过了就被阻塞，在等待队列中呆着呢。话说js是单线程的，也不存在真正的并发，所谓的并发请求都是模拟出来的。

我网上一搜，全是写这个的，才知道这是字节跳动的一道经典面试题。考察队列、promise、递归的运用。所以我就自己写这么一个东东。今天写的是一个升级版，可以像Promise.all一样，按参数顺序获取fulfilled的结果数据。没有网上那么花哨，什么promise.settleAll、promise.race这些api都没有使用，只使用最基本的方法实现，话不多说，上源码：


```
// 模拟请求
function request(url){
    return new Promise((resolve, reject) => {
        let t = setTimeout(() => {
            resolve(url)
        }, 2000)
    })
}

function requestUnderMax(urls , requestFn, max) {
    return new Promise((resolve, reject) => {
        let i = 0;          // 一个i对应一个fulfilled结果, 让最终结果是有序的
        let isRejected = false
        let queue = [];     // 只能放num个promise成员
        let resArr = Array(urls.length);     // 存放fulfilled结果，有顺序的
        let error = false;
        function loop(urls) {
            // 一旦error是true，说明前面某个promise rejecte了。
            if (error) return;
            // 递归边界条件，urls处理完了，length === 0
            if (urls.length === 0 ) {
                Promise.all(queue).then(() => {
                    resolve(resArr);
                })
                return
            }
            if (queue.length < max && !error) {
                let p = requestFn(urls.shift())
                .then(res => {
                    // 1. 根据索引，记录结果
                    // 2. 如果urls还有没处理的，那么让当前promise从queue中出栈;
                    resArr[p.index] = res;
                    if (urls.length > 0) {
                        queue.splice(queue.indexOf(p), 1)
                        loop(urls)
                    }
                })
                .catch((err) => {
                    error = true
                    reject(err)
                })
                p.index = i;
                // 每次将promise添加到队列之后，就自增，带遍处理下一个
                i++;
                queue.push(p)
                return loop(urls)
            }
            return
        }
        loop([...urls]);
    })
}


let urls = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13];
requestUnderMax(urls, request, 8).then((res) => {
    // 期望结果顺序一直， res [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13]
    console.log('res', res)
})
```

#### 解读
考虑到易用性，并发控制函数返回的是一个promise。对于通用性考虑，第二个参数是一个返回promise的请求方法。当然第一个参数也可以改造成参数数组。

在实现过程中运用了尾递归优化，对调用栈很友好。对于递归边界条件的控制，放在了递归函数的头部，这是重中之重。对于异常也有响应处理，如果有一个请求发生错误，直接执行reject，并且停止递归。

美中不足是递归函数有副作用，对于error变量，它在外层作用域下。但在loop递归函数中，却可以改变其值，这样不太好。可以考虑将其放在loop函数参数的第一项，这样也满足错误优先处理的原则