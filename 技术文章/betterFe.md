#### 1. 网页性能指标
##### 1. 分析数据收集

> 为何要最先谈性能指标？因为这是定义一个web页面性能好坏的标准，没有标准，谈何优化？前端优化几乎全部都是针对于这些指标的。这些指标的关键数据，直接或间接地影响着业务增长，比如白屏用时、首屏用时、用户可操作时间点，这些会最终影响一个产品的用户留存，所以必须熟悉这些指标。

借助浏览器performan.timing对象
```
const {
    domainLookupStart,   // dns开始查询的时间戳
    domainLookupEnd,     // dns结束查询的时间戳
    connectStart,        // 开始tcp握手的时间戳
    connectEnd,          // 结束tcp握手的时间戳
    navigationStart,     // 浏览器开始卸载前一个页面的时间戳(用户输入url),
    requestStart,        // 第一个http请求 请求开始的时间戳
    responseStart,       // 第一个http请求 响应开始的时间戳
    responseEnd,         // 最后一个http请求 响应结束的时间戳
    domLoading,          // 开始解析dom树的时间
    domComplete,         // dom树解析完成的时间戳(dom中的资源也下载完毕)
    domInteractive,       // dom树解析完成的时间戳(并没有加载完dom树中一些资源)
    domContentLoadedEventEnd,   domContentLoaded事件回调函数执行完毕(可以理解为内部的脚本也执行完了)
    loadEventEnd,        // load事件执行完毕的时间
} = window.performance.timing;
```

1. DNS查询耗时 ：domainLookupEnd - domainLookupStart
2. TCP链接耗时 ：connectEnd - connectStart
3. request请求耗时 ：responseEnd - requestStart
4. 解析dom树耗时 ： domComplete - domLoading
5. domready时间(用户可操作时间节点) ：domContentLoadedEventEnd - navigationStart
6. 总下载时间 ：loadEventEnd - navigationStart
5. 白屏用时 ：responseStart - navigationStart
6. 首屏用时：从用户在地址栏输入url-> 卸载当前页面 -> 白屏 -> 屏幕可视区出现任何新页面的元素，这期间的用时，计算非常复杂;

##### 首屏时间统计
1. 通过MutationObserver Api, 监听整个document的变化。在每次dom发生改变的时候，筛选出，body内部的所有外层元素, 并记录下用时；
2. 用performance.getEntriesByType('resource')，筛选出所有图片资源的加载情况，如果开始加载的时间点在通过MutationObserver最后一次变化的时间之前，而图片响应时间，在该时间之后，说明这个图片的响应结束时间点就是首屏时间的最后时间点。

```
   function getFirstScreenTime() {
        const outerElementMutations = [];   // 收集多次document内部dom元素更新，更新的时间，更新了哪些外层元素
        const ignoreEleList = ['script', 'style', 'link', 'br'];
        const isOuterElement = (element, collectedOuterElements = []) => {
            if (element === document.documentElement) {
                // 直到递归到顶层元素都没被收集，说名是一个需要被收集外层元素
                return true;
            }
            else if (collectedOuterElements.indexOf(element) !== -1) {
                // 说明没被收集过
                return false;
            }
            else {
                // 不是顶层元素，也没在数组中，继续向父元素查找
                return isOuterElement(element.parentElement, collectedOuterElements);
            }
        }
        return new Promise(function (resolve, reject) {
            var observeDom = new MutationObserver(mutations => {
                var currentMutationInfo = {
                    during: performance.now(),
                    roots: []
                };
                let tempElements = []
                mutations.forEach(currentMutation => {
                    // 遍历已经添加到dom的元素类型节点

                    const addedElements = Array.from(currentMutation.addedNodes).filter(ele => ele.nodeType === 1 
                    && ignoreEleList.indexOf(ele.nodeName.toLocaleLowerCase()) === -1);

                    addedElements.forEach((ele) => {
                        // 通过打印顺序，可以看出dom的挂载按照深度优先

                        if (isOuterElement(ele, currentMutationInfo.roots)) {
                            currentMutationInfo.roots.push(ele);
                        }
    
                    });

                });

                if (currentMutationInfo.roots.length) {
                    outerElementMutations.push(currentMutationInfo);
                }
            });

            // 监听document子dom树变化
            observeDom.observe(document, {
                childList: true,
                subtree: true
            });

            // 五秒后解除观察者
            let t = setTimeout(() => {
                observeDom.disconnect();
                resolve(outerElementMutations);
                clearTimeout(t)
            }, 5000);
        }).then(outerElementMutations => {
            // 选出时间最长的dom变化详情(通过异步请求，加载脚本后生成dom，时间是最长的)
            let [ MaxDuringTime, ...rest ] = outerElementMutations.sort((a, b) => b.during - a.during);
            let imgResorces = performance.getEntriesByType('resource').filter(resource => resource.initiatorType === 'img')
            imgResorces.forEach(resource => {
                const { fetchStart, responseEnd } = resource;
                /* 
                如果开始请求的时间在MaxDuringTime，之内，证明这些图片是首屏请求的dom中的一部分。
                如果这些图片响应完成的时间段，大于其他主要元素加载到dom上的时间，那么认为这个图片拖后腿了。
                首屏时间为这个图片加载完成的时间，接着挨个比较有没有比这个图片加载时间更长的图片
                */
                if (fetchStart < MaxDuringTime && responseEnd > MaxDuringTime ) {
                    MaxDuringTime = resource.responseEnd;
                }
            })
            return MaxDuringTime;
        });
    }

    getFirstScreenTime().then((res) => {
        console.log('firstScreenInfo', res)
    })

```

> 这个方法能相对准确的记录首屏幕时间，必须在head标签内部使用，否则没等开启观察者，就结束了，就没什么意义了。

> 这个方法有一个bug，就是如果有一些外层元素高度超出视口高度，那么首屏时间就会不那么精确，比真正的视口区域首屏加载时间要长。但是由于此时视口内部的dom还没有加载，无法判断哪些element节点是在视口内的。所以此方法只能最大程度统计，存在一定误差

> 针对上面的误差，其实有方法弥补，可以通过innerSectionObserver Api，对外层元素进行观察，只有进入到可视区了，才进行真正渲染，没有进入可视区，只当做一个普通div，这样不仅统计变得精确了，而且很大程度提升了首屏幕渲染时间。为此，我写过一个React的hoc，并在工作中使用了它
```
import React from 'react';
import { generateUUIDV4 } from 'utils';

export default (opts = {}) => Component =>
class extends React.PureComponent {
    constructor(props) {
        super(props);
        // 进入可视区域加载真实组件
        this.state = {
            loaded: false,
        };
        // 用于查找dom，绑定观察者
        this.key = generateUUIDV4();
    }

    componentDidMount() {
        // 监视器回调函数
        const callback = (entries) => {
            entries.forEach((entry) => {
                // 进入可视区域 & 未被加载过
                const { loaded } = this.state;
                if (entry.isIntersecting && !loaded) {
                    // 进入视口，加载真实组件
                    this.setState({
                        loaded: true,
                    });
                }
            });
        };
        // 监视器配置参数
        const options = {
            root: opts.root || null,
            rootMargin: opts.rootMargin || '0px 0px 10px 0px',
            threshold: opts.threshold || [0],
        };
        // 创建监视器实例
        this.observer = new IntersectionObserver(callback, options);

        // 监视虚拟Dom
        this.observeDom = document.getElementById(this.key);
        this.observer.observe(this.observeDom);
    }

    componentWillUnmount() {
        // 移除监视Dom
        this.observer.unobserve(this.observeDom);
        this.observer.disconnect();
    }

    render() {
        const { loaded } = this.state;
        return loaded ? <Component {...this.props} /> : <div id={this.key} style={{ height: 100 }}></div>;
    }

};

```
> 只要用这个高阶组件包裹某个元素，就可实现懒渲染的效果。

##### 2. 分析数据上报
**1. Image Get Request**
> 发送get请求，不会跨域
```
let i = new Image();
i.src = 'XXX'
i = null
```
**2. Beacon Api Post Request**
> 信标，专门用于上报分析数据的api。navigator.sendBeacon(url, data),data要求是formData格式；
```
navigator.sendBeacon(url, formData)
```

> 向下兼容的上报方法的实现
```
function sendAnalysisInfo (url, data = {}) {
    if (typeof data !== 'object' || data === null) {
        throw new Error('data type error')
    }
    if (navigator && navigator.sendBeacon) {
        let formData = new FormData();
        for(let [key, value] of Object.entries(data)) {
            formData.append(key, value)
        }
        navigator.sendBeacon(url, formData)
    } 
    else {
        let paramStr = ''
        for(let [key, value] of Object.entries(data)) {
            paramStr += `&${key}=${value}`
        }
        paramStr = paramStr.splice(1)
    }
    return;
}
```

**3. 伴随业务接口，一起发送。**
> 将分析数据进行包装，在请求之前用拦截器判断是否发送过性能分析数据，如果没发送过，就提前封装进接口的数据内。在没有独立的监控处理服务器情况下，会这么干，以减少网络消耗。


##### 3. 数据处理
> 服务端同一时间可能会发生大量请求上报的信息。通常会批量处理这些上报数据，通常会以一个队列的数据结构收集这些请求信息。为了数据更加有代表性，通常会取大量数据的平均值。将队列的check上限定为2000，当收集满了数据，就取其平均值，这个平均值作为与我们设定的参考阈值的对比数据。如果是nodejs，切记不能使用普通的数组当作队列！！！搞不好会造成堆内内存爆栈，别问我咋知道的。。。。。可以使用buffer，不过也要保证收集数据后，先出栈，再入栈，形成增量统计

> 做平均值处理之前还要经过脏数据清理。举个例子: 一组数据arr = [5, 6, 5, 5, 6, 4, 5]，新过来一个10，这个10就是脏数据，假设我们设定+-2作为脏数据判断的阈值，10 不在arr平均值正负2的范围内，所以不能纳入合法统计数据。

> 数据处理过程比以上讲的复杂的多，通常由公司内部数据团队，专门提供算法，有开发人员具体实现。当收集到分析数据后会写入数据库。然后再性能分析平台，以特定的图表进行展示。很多系统会将数据分析接入告警系统。计算到的数据超过预定阈值，发邮件告警。。。。。


#### 2. 网络层面优化
> 目前来看，pc、移动端的设备性能已经非常优秀了，其浏览器内核也有了很大的进步。渲染效率普遍非常高。前端性能最大的瓶颈在网络io、渲染方式上。

**1. DNS预解析**
> 对于静态资源很多的站点，会配置多个dns服务器，以实现最高并发量。但是第一次加载页面加载静态资源时，免不了对cdn域名进行解析。配置了dns预解析后，可以在解析到加载静态资源的标签之前，提前解析cdn域名对应的ip地址。减少网络io时间。

```
// 开启dns预解析
<meta  http-equiv="x-dns-prefetch-control" content="on">
// 配置要预先解析的域名
<link  rel="dns-prefetch" href="//www.qiniuyun.com">
```
**2. 采用DNS服务器加载静态资源**
> 如果一个站点服务，请求量级很大，那么用cdn服务器减轻站点服务的负载，是一个很好用的办法。cdn服务器通常内置了对http缓存的设置，包含一系列强缓存、协商缓存、回源策略等配置。

> 现代浏览器通常允许的TCP连接并发数，在一个域下为6。如果有大量的静态资源最好配置多个不同域名的cdn。


**3. 如果可以的话，采用http2加载资源**
> http2默认https，更安全。二进制分帧传输，比1.1的字符串效率高。头信息进行压缩，体积小；多路复用，一个tcp连接下，多个请求只要是一个域名下，可以复用连接。性能特别好！

**4. 对于静态资源开启http compress gzip压缩**

**5. 小图片合成雪碧图，减少请求数量，降低n-1个网络负载**

#### 3. 静态资源优化
**1. 使用合适的多媒体(图片、视频)**
> png格式图片采用无损压缩、jpg格式图片采用有损压缩但体积更小。对于移动端来说，要尽量避免高清图片。节省用户流量，提高加载速度。通常图片本身也会被cdn厂商压缩，又经过http压缩，所以图片质量把控，在移动端开发很重要

> webP，如果环境是谷歌内核浏览器或者webview，可以采用webP格式图片，这种格式图片同时支持有损压缩、无损压缩、透明。压缩效率非常高，可以节省网络带宽；

> 对于移动端的开发，很多情况下采用背景图，可以通过css媒体查询dpi，决定采用2X或者3X图片。加载更高清的图片，提升体验。

> 视频最好采用推流模式，可以提升加载观看体验

**2. 移动端用字体图标代替雪碧图**
> svg矢量字体图标在不同dpi设备下，不会失真。而雪碧图就不行了

**3. Html、css、js的压缩**
> 生产环节，html、css、javascript都应该被压缩，减少体积就能较少response时间


#### 4. 渲染层面优化
**1. 避免书写阻塞渲染的代码；**
1.1 为何要将css放入head内？
css会阻塞浏览器的渲染，不会阻塞dom的解析。css解析成cssom后在将解析完成的domtree合并，形成渲染树。如果css不放在前面，很可能渲染出来的内容没样式，而后才有样式。所以css阻塞是可以被利用的。
1.2 js阻塞
javascript内封装了大量的可以操作dom的api，提供了控制渲染的能力。所以当js线程在执行状态时，GUI渲染线程会被挂起。为了避免脚本阻塞渲染。script通常会被放在body的底部,或者使用deffer、async属性的script
1.3 脚本中不要写docuemne.write、alert这样的阻塞的脚本
```
<script deffer src></script>
deffer属性的标签，内部的脚本会在DOMContentLoaded的回调中执行，不会立即执行，也就不会阻塞GUI渲染线程，所以有defer可以大胆放在head内。

async属性的标签，内部的脚本会在脚本资源下载完成后立即执行，有阻塞渲染的风险
```
##### requestIdleCallback(callback)

> 如果有可能会阻塞渲染的计算相关操作，放到requestIdleCallback内执行。当浏览器渲染每一帧没用上16.6ms，有时间空闲，就会执行回调，这样不会阻塞渲染。这个回调函数执行之间最长为50ms。react就是按照这个原理，有一套自己对requestIdleCallback的实现。

**2. 尽量避免重排；**
> Layout 布局，浏览器合成renderTree后的下一个步骤。就是计算各个元素的位置，元素之间的关系。
paint 绘制，简单理解为给元素上色就好。
Repaint 重绘，重新绘制，理解为更改颜色就行。
Relayout 重排，也叫回流Reflow，就是重新布局。重排会伴随重绘。重新计算各元素尺寸，位置，对应关系，非常耗费性能。

> 主流浏览器刷新率为60fps，就是每一帧的时间为16.6ms，如果渲染这一帧超过这个时间，就会卡顿。单位时间密集度过高的重排就会造成卡帧。

会触发重排（回流）的操作：

**2.1. 修改元素盒子模型相关属性：**
> width/height、margin、padding、border-width、display

**2.2. 修改元素位置、布局方式**
> positon、display、float

**2.3. 修改元素内容**
> font相关属性、text-align、vertical-align、line-height、overflow

**2.4. 获取元素的宽高、位置、计算属性**
> 调用getComputedStyle()，读取元素width、height、client相关属性、offset相关属性、scroll相关属性
**2.5. 浏览器触发scroll、resize等事件；**

##### 如何避免重排或者降低重排带来的影响？
> 使用现代框架，不直接操作dom，通过diff算法比对vdom树，将重排范围降到最低；

> 使用防抖/节流函数，降低回流频率，给重排留足充足的时间

**防抖：不让连续触发的目标函数一直执行，只有目标函数连续触发，停止一定时间之后，才会执行一次
使用场景：1. 表单校验，只有用户停止输入300s内没继续操作，才进行校验，2. 搜索联想，只有用户停止输入300s内没继续操作才可以通过接口发请求**

**节流：不让连续触发的目标函数执行过快，让其每次执行之间有固定间隔。
使用场景：滚动懒加载，降频请求接口**
```
        // 防抖实现
        function debounce(fn, delay) {
            // 此处不可以使用箭头函数，否则这个位置的匿名箭头函数是undefined
            let timer;
            return function (args) {    
                // this为调用函数的对象，通常为dom对象
                /* 连续触发事件，每次都要清空定时器，让之前的延时作废，重新生成定时器，
                直到有足够的延迟时间且没有再次触发事件，这时候timer中的fn才会真正执行
                */
                clearTimeout(timer)
                //此处需要使用箭头函数，让this是函数的调用者(通常是dom)
                timer = setTimeout(() => {     
                    fn.call(this, args)
                }, delay)
            }
        }
        
        // 节流实现
        function throttle (fn, delay) {
            let canRun = true, timer;
            return function(args){
                // 如果当前定时器还在挂起状态，任务还没执行，则直接返回
                if (!canRun) return;
                canRun = false;
                timer = setTimeout(() => {
                    fn.call(this, args)
                    // 只有到时间间隔了，才可以触发并生成下一次的延迟执行
                    canRun = true;   
                    clearTimeout(timer)
                }, delay)
            }
        }


```
> 使用requestanimationframe(), 将引起重排的操作，尽可能合并在一次重排之内。减少单位时间的重排次数

> 如果有动画方面需求，不使用改变dom位置的，使用transform实现。transfrom内的偏移，全部都在一个独立的图层内进行，会将所有重排锁定在这个图层内部，不会引发全局的重排。

**3. 网页内容懒加载**
> 按需加载，可以大大减少首屏渲染时间。以下是两个常见案例：

> 滚动懒加载图片、高阶组件+interSectionObserverApi懒加载dom结构。

> 借助于webpack，对其他路由页面的脚本代码与样式通过import异步函数打包成单独chunk，懒加载。

Demo参考：https://github.com/zhm19901224/Blogs/blob/master/%E6%8A%80%E6%9C%AF%E6%96%87%E7%AB%A0/IntersectionObserver.md

**4. 带宽空闲时进行预加载**
```
document.onreadystatechange = () => {
    // complete代表文档和子资源都加载完成，代表目前带宽空闲
    if(document.readyState == "complete"){
        // 预加载其他页面的资源
    }
}
```


5. Server Side Render服务端渲染
> 随着高性能响应式前端框架及单页应用SPA的普及，前端开发效率大幅提升，但也带来了一系列问题。

CSR过程:

用户输入网站URL->服务器返回html->html解析到script->下载打包好的框架代码与业务代码
->执行业务代码，请求接口数据->向页面填充内容

> 页面正式渲染，排在了下载业务代码、执行代码、请求接口数据之后，这导致了很长的白屏时间、并且搜索引擎爬虫很难抓取到网页内容SEO很差。所以又出现了传统的SSR渲染方式

> 用户输入网站URL -> node端请求api服务器的接口 -> node端运行框架代码将接口数据拼装成hmtl字符串 -> 将html返回给前端。前端先展示页面，再执行框架、业务脚本，绑定事件，绑定其他页面路由。

> 这么做改善了SEO，也大大缩短了白屏时间、首屏加载时间。衍生出了node中间层架构，不仅用于服务端渲染、也用于转发客户端接口，提前处理好接口响应数据再返回给客户端。这么做好处很多，但也一定程度加大了复杂度、人力成本，通常pc端门户网站、运行在app中的webview会采用这种方式，管理系统则不会采用这种架构。


#### 5. 代码层面优化
**1. ESLint、TSLint，配置规范化的ESLint，可以做到团队内部代码风格最大程度统一。**

**2. 慎用闭包、IIFE，时刻注意加载到Dom这类型的内存泄露。**

**3. 全局变量小心使用，最好用__XXX__别名形式命名。**

**4. 尽量遵守，SOLID开发原则。尤其是单一职责原则，开放封闭原则；**

#### 其他
**1. 移动端最好强制开启GPU加速，提高动画流畅性**
```
webkit-transform: translateZ(0);
-moz-transform: translateZ(0);
-ms-transform: translateZ(0);
-o-transform: translateZ(0);
transform: translateZ(0);
```

**2. PWA 渐进式web应用**
> 采用web Work，与本地缓存。在无网条件下，用户依然可以浏览缓存的页面。极大提高体验性

**3. 移动端开发点击事件问题**
> 移动端点击事件使用最好使用touchstart，a链接用脚本方式代替(移动端基本不用考虑seo)。这样既避免了click事件，在移动端中为了判断双击放大而造成的300ms延迟。又避免了a链接点击穿透。不要采用fastclick，它存在点击穿透的bug。

**4. css 选择器尽量扁平化，不要选择器层级写得过长，这样会增加解析时查找的时间**

**5. 如果采用了很老的dom操作库，手动在dom上绑定事件，务必采用事件委托。节省内存资源，避免新增子节点后，事件未绑定的尴尬；**



#### 最后

参考：https://cloud.tencent.com/developer/article/1650697

> 想不起来还有哪些优化的点了，过后会陆续补充。
> 码字不易，转载请注明出处及作者。谢谢!