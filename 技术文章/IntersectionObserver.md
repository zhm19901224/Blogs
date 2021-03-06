#### 简介
> IntersectionObserver用于监控目标元素在可视区，可见行的变化。由于其是异步执行，而且会在渲染空闲状态下运行，所以性能非常好，可以替代手动对scroll事件的监听了；

> 具体用法参考：[https://developer.mozilla.org/zh-CN/docs/Web/API/IntersectionObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/IntersectionObserver)
#### 1. 图片滚动按需加载

现在有如下dom需要按需加载
```
    <ul class="image-container">
        <li>
            <img src="" class="image-item" data-url="https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=2853553659,1775735885&fm=26&gp=0.jpg">
        </li>
        <li>
            <img src="" class="image-item" data-url="https://ss1.bdstatic.com/70cFuXSh_Q1YnxGkpoWK1HF6hhy/it/u=1870521716,857441283&fm=26&gp=0.jpg">
        </li>
        <li>
            <img src="" class="image-item" data-url="https://ss1.baidu.com/-4o3dSag_xI4khGko9WTAnF6hhy/image/h%3D300/sign=4ac31a85b5389b5027ffe652b534e5f1/a686c9177f3e6709b52f456437c79f3df8dc5579.jpg">
        </li>
        <li>
            <img src="" class="image-item" data-url="https://ss0.baidu.com/-Po3dSag_xI4khGko9WTAnF6hhy/image/h%3D300/sign=a9201e46ecfe9925d40c6f5004a95ee4/d53f8794a4c27d1e41123d4717d5ad6eddc43878.jpg">
        </li>
        <li>
            <img src="" class="image-item" data-url="https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=121352583,3553479540&fm=26&gp=0.jpg">
        </li>                        
    </ul>
```
在IntersectionObserver Api出现之前，我们需要让图片按需加载该怎么做的？
```
const viewHeight = document.documentElement.clientHeight;
function lazyload () {
    window.requestAnimationFrame(() => {
        const imgElements = document.querySelectorAll("img[data-url]")
        Array.from(imgElements).forEach((item, index) => {
            if (!item.dataset.url) return;
            let { top } = item.getBoundingClientRect();
            if (top < viewHeight) {
                item.src = item.dataset.url
                item.removeAttribute('data-url');
            }
        })
    })
}
lazyload();
document.addEventListener('scroll', lazyload)
</body>
```
> 必须调用window.requestAnimationFrame(callback)，下次重绘之前执行回调，避免短期内产生大量重排

现在的做法：
```
const io = new IntersectionObserver((changes) => {
    changes.forEach(item => {
        let imgTarget = item.target;
        if (!item.isIntersecting) return;
        imgTarget.src = imgTarget.dataset.url
        io.unobserve(imgTarget)
    })
});
const imgElements = document.querySelectorAll("img[data-url]")
Array.from(imgElements).forEach(item => io.observe(item))
```

> 1. 代码变得更加精简了，10行代码就实现了！不用移除元素属性，不需要通过判断是否有标志属性，区分是否加载过。只要触发了进入视口，直接解除观察者，十分优雅。2. 性能很好，由于是异步的，会在GUI渲染空闲执行，不需要手动写requestAnimationFrame在重回前执行dom操作了。太省心了！

#### 2. 无限滚动列表

```
<!DOCTYPE html>
<html lang="zh-cn">
    <head>
        <meta charset="utf-8">
        <meta http-equiv="x-ua-compatible" content="IE=edge, chrome=1">
        <title>无限滚动列表</title>
        <style type="text/css">
        	* {
        		padding: 0;
        		margin: 0;
        	}
        	.list-container {
        		list-style: none;
        	} 
        	.list-item {
				width: 100%;
        		line-height: 120px;
				text-align: center;
				border: 1px solid #ddd;
				border-top: none;
        	}
        </style>
    </head>
    <body>
        <ul class="list-container" id="container">
			<li class="list-item">1</li>
			<li class="list-item">2</li>
			<li class="list-item">3</li>
			<li class="list-item">4</li>
			<li class="list-item">5</li>
			<li class="list-item">6</li>
			<li class="list-item">7</li>
			<li class="list-item">8</li>
			<li class="list-item">9</li>
			<li class="list-item">10</li>                      
        </ul>
		<script type="text/javascript">
			let currentPage = 1;
			let pageSize = 10;
			const io = new IntersectionObserver(changes => {
				changes.forEach(item => {
					let element = item.target;
					if (!item.isIntersecting) return;

					io.unobserve(element);
					requestMoreData().then((data) => {
						data.forEach((item) => {
							let li = document.createElement('li')
							li.innerText = item
							li.className = 'list-item'
							container.appendChild(li)
						})
						registObserver(1)
					})
				})
			})
			
			const getListItem = (sel) => Array.from(document.querySelectorAll(sel));
			const registObserver = (lastNum) => {
				// lastNum是倒数第几个，为lastNum元素进注册观察者
				let listItems = getListItem('.list-item')
				io.observe(listItems[listItems.length - lastNum])
			}

			registObserver(1);
			// 模拟请求数据
			const requestMoreData = () => {
				let start = currentPage * pageSize + 1
				let data = [];
				for(let i = start;i < start + pageSize; i++){
					data.push(i)
				}
				currentPage = currentPage + 1;
				return Promise.resolve(data)
			}

        </script>
    </body>
</html>
```

> 节流函数都不用管了，也不需要处理频繁重排。特省事！


#### 3. 无侵入埋点脚本，对曝光事件处理

```
<!DOCTYPE html>
<html lang="zh-cn">
    <head>
        <meta charset="utf-8">
        <meta http-equiv="x-ua-compatible" content="IE=edge, chrome=1">
        <title>无限滚动列表</title>
        <style type="text/css">
        	* {
        		padding: 0;
        		margin: 0;
        	}
        	.list-container {
        		list-style: none;
        	} 
        	.list-item {
				width: 100%;
        		line-height: 120px;
				text-align: center;
				border: 1px solid #ddd;
				border-top: none;
        	}
        </style>
    </head>
    <body>
        <ul class="list-container" id="container">
			<li class="list-item">1</li>
			<li class="list-item">2</li>
			<li class="list-item">3</li>
			<li class="list-item">4</li>
			<li class="list-item">5</li>
			<li class="list-item">6</li>
			<li class="list-item">7</li>
			<li class="list-item">8</li>
			<li class="list-item">9</li>
			<li class="list-item"  data-report='type:exposure;eventId:1'>10</li>                      
        </ul>
		<script type="text/javascript">
			document.addEventListener('DOMContentLoaded', () => {
				const io = new IntersectionObserver((changes) => {
					changes.forEach(item => {
						let elem = item.target;
						if (!item.isIntersecting) return;
						io.unobserve(elem)
						let arr = elem.dataset.report.replace(/\s*/g,"").split(';')
						let info = {};
						arr.forEach(item => {
							let [key, value] = item.split(':')
							info[key] = value;
						})
						
						info.time = Date.now()
						report(info)
					})
				})
				const eventTargets = Array.from(document.querySelectorAll('[data-report]'));
				// 在所有买点元素中，过滤出曝光埋点元素
				const exposureElements = eventTargets.filter(element => {
					return element.dataset.report.replace(/\s*/g,"").indexOf('type:exposure') !== -1
				})
				// 为曝光埋点元素注册观察者
				exposureElements.forEach(elem => {
					io.observe(elem)
				})
				
			});
			// 发送请求，上报触发的埋点信息
			function report(info){ console.log(info) }
        </script>
    </body>
</html>
```


#### 总结
以上三个案例，是本人在实际工作中所遇到的几个情景。非常实用。通过这三个案例，我们清楚了，IntersectionObserver的优点。以及传统方式用scroll的缺点。

码字不易，如需转载，请注明出处，多谢！