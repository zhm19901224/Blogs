### 前言
> 记得两年前做过一个项目，需要通过管理后台上传mp4文件，我对接的后端是做php。当时出现一个状况，当时网络条件不是很理想，一个4Gb大小的mp4在上传到服务端后，Content-Type丢失了，数据包也不是完整的。

> 一桶雾水过后，网上一搜，知道这种大文件需要通过分片上传来实现。那时个人技术水平有些萌新，只能找一些别人写好的工具拿来用。因为公司有买七牛云的云存储，做静态资源服务器（公司财大气粗啊）。于是就选用了七牛云上传相关的JSSDK。要不说那时候菜呢，实现完了就完了。也没去关心内部的实现原理。

> 两年后在一家某二手车公司工作的我，需要上传一些静态资源，这时候去看了一下公司静态资源服务的文档，准备开用，发现上传功能挂了。。。。。。

> 于是好好研究了一下前端上传的各种手段，决定从零出发，写一个基于nodejs的上传demo，涵盖各种各样主流的上传方式。为了加深对nodejs的理解，决定不用任何第三方library，从零开始。这样有自己的设计，而且还能体会造轮子的快乐！原谅我啰嗦完了。。。。。

#### 多文件异步上传原理
前端任务
1. input type="file"的上传表单上绑定change事件。
2. 在change事件回调函数中，通过target参数, target.files拿到上传的文件数组。
3. 实例化formData对象，遍历files数组，以索引为key，file为value，append到formData对象中。将formData传递给请求的方法
```
function uploadHandle(){
    var formData = new FormData();
    for(let i = 0;i < uploadFiles.length;i++){  
        formData.append(i, files[i])
    } 
    simpleSendData(formData)
}
```
4. 准备请求方法，实例化xhr对象，将formData放入请求body中发送。

后端任务
1. 处理post请求，拿到完整的数据包Buffer对象；
2. 解析Buffer对象，解析出多个文件Buffer；
3. 将每个buffer对象写入到目标文件中；

#### 拖拽上传原理
前端任务

1. 绘制好拖拽上传的触发区域。将区域元素绑定drop事件
2. 让document 的drop事件触发时候，取消默认事件，防止下载的出现；
3. 处理下载区域drop事件, 在回调函数中通过e.dataTransfer.files,拿到药下载的文件。剩余逻辑和普通上传一样；
```
document.addEventListener('drop', function(e){
    e.preventDefault();
}, false)
document.addEventListener('dragover', function(e){
    e.preventDefault();
}, false)

let dragArea = document.getElementById("dragArea")
dragArea.ondrop = function(e) {
    previewFiles(e.dataTransfer.files);
    uploadFiles = e.dataTransfer.files;
    e.target.innerHTML = '请将文件拖拽到此区域';
    uploadBtn1.style.display = "block"
}
```
#### 文件预览原理
假设文件就是图片
1. fileReader类，遍历文件数组，通过实例化fileReader读取文件，在读完之后将文件读取成base64

2. 创建图片元素设置src属性，挂载在dom上。
```
function previewFiles(files) {
    let previewContainer = document.getElementById('previewContainer')
    Array.from(files).forEach(file => {
        let previewContent;
        let fileReader = new FileReader();
        fileReader.onload = (e) => {
            let img = new Image()
            img.src = e.target.result;
            img.className = "previewImg center"
            previewContainer.appendChild(img)
        }
        fileReader.readAsDataURL(file, "UTF-8");
    })
}
```

#### 分片上传原理
前端任务
1. 生成分片：先拿到文件的大小、定义好每一分片的大小，利用Blob.prototype.slice api，将file文件进行切割，切割后以索引命名(方便按顺序合并)。在将切割完的blob转成file类，添加到formData中
2. 将所有分片，并发发送请求到后端。
3. 当所有请求的promise状态都为fulfilled时，给后端发送请求，将文件名、文件的内容哈希，发送给后端，让后端将所有文件分片按照顺序合并，合并后计算出内容hash，和请求的内容hash做一致性对比.

后端任务
1. 解析请求body的formData，拿到每一个分片，将分片buffer数据写入到临时目录中，依然按照原分片名称命名。
2. 收到前端合并请求后，按照分片名称(索引)合并，合并到静态资源服务器目录files下，合并后检查文件内容hash，看是否和前端的传来的内容hash一直，如果一直，将文件地址返回给前端。并且要清除temp临时文件夹下面的chunk

#### 续传、秒转原理
1. 秒传原理：

后端在分片上传基础上，每次在合并验证完完分片后，要将文件索引，文件名记录起来。（应该写入redis或者json文件，我图省事写在变量内存中了）

前端在上传之前，先发送一个预请求，将文件内容hash、文件名给后端，后端收到请求去检查上传记录，发现索引一致，直接将文件位置发送给前端，告诉前端不要上传了。

2. 续传原理

前端给后端发预请求，后端发现没有上传完成记录，但是临时文件下有很多chunk分片。说明没上传完。这时候需要计算出最大索引，告诉前端需要从哪个索引的分片继续上传。

前端收到继续上传的响应后，从继续的索引处计算分片。继续发送上传的post请求


#### 设计与开发

1. 环境搭建：定义package.json的执行脚本，必须安装nodemon或者supervisor这样的文件watch工具，要不然每次改代码还要重新启动node进程；
2. 定义文件目录、代码结构; 思考：先要实现一个简单的node server，它需要支持上传页面的站点服务，上传文件相关的一系列接口服务。所以最好以传统的mvc模式进行开发。route只处理路由，template下html写前端逻辑。controller负责具体业务逻辑，并且连接数据层(上传记录)
3. 按照原理，画逻辑流程图或者UML类图
4. 根据类图，统计出要实现的所有方法，和轮子，定义好所有要写的方法；
5. 用定时器模拟分片发送慢的场景，用暂停按钮模拟断网以测试续传；准备好要测试的脚本和文件；
6. 正式业务代码编写(期间不停的修补逻辑流程图的漏洞)；
7. 优化代码；对重复逻辑进行封装；

#### 遇到的坑

1. node端解析post请求formData数据，完全不想content-type: application/json那么容易。尤其是注入了二进制文件之后，解析尤其困难。有现成了multiparty库可用，但是开发之前立flag不用第三方模块了，就只能自己造轮子。将formData的buffer转字典。为此Buffer原型还没有slite方法，只能自己写polyfill;
```
// 为Buffer添加拆分的方法
Buffer.prototype.split = function (boundary){
    let arr = [];
    let cur = 0;
    let n = 0;
    while((n = this.indexOf(boundary,cur)) != -1){
      arr.push(this.slice(cur,n));
      cur= n + boundary.length
    }
    arr.push(this.slice(cur))
    return arr
}
// 将formdata里面全部的name对应关系，转换成字典
const formDataBufferParser = (req, buffer) => {
    let res = {};
    let str = req.headers['content-type'].split('; ')[1]
    let boundary = '--'+str.split('=')[1]
    let bufferArr = buffer.split(boundary)
    bufferArr.shift()
    bufferArr.pop()
    bufferArr = bufferArr.map(buffer => buffer.slice(2, buffer.length - 2))
    // formData append几项目 就有继承成员
    bufferArr.forEach((buffer, index) =>{
    // 判断是不是文件数据
        let n = buffer.indexOf('\r\n\r\n');
        let disposition = buffer.slice(0,n).toString();
        let content = buffer.slice(n + 4)
        if(disposition.indexOf('\r\n') !== -1) {
            // 当前 buffer存储的是文件
            let [line1, line2] = disposition.split('\r\n'); // line1 Content-Disposition: form-data; name="file"; filename="0"
            let [, name, filename] = line1.split('; ');
            name = name.split('=')[1]
            name = name.slice(1, name.length - 1);
            filename = filename.split('=')[1]
            filename = filename.slice(1,filename.length-1);
            let mime = line2.split(': ')[1]
            res[name] = {
                filename,
                mime,
                file: content
            }
        } else {
            // 当前buffer存储的是文本信息
            let [line1] = disposition.split('\r\n');
            let [, name] = line1.split('; ');    // name="aaa"
            name = name.split('=')[1]
            // 去掉引号
            name = name.slice(1, name.length - 1) 
            res[name] = content
        }
    })

    return res
}
```

2. 合并文件的时候，只想着拿着分片paths数组，循环创建可读流，管道流流入到可写流里就行了。我年轻了，报错提示内存泄露。网上搜了一下相关案例。需要手动控制可读流、可写流的关闭。为此单独提炼了一个合并文件的方法
```
// 将多个chunk文件，合并到目标文件中去
const mergeFiles = (filePaths, targetFilePath) => {
    return new Promise((resolve, reject) => {
        let writeStream = fs.createWriteStream(targetFilePath)
        writeStream.on('finish', () => {
            console.log('mergeFinish')
            resolve([null, 'ok'])
        })

        function merge(paths){
            if (!paths.length) {
                return writeStream.end();
            }
            let readPath = filePaths.shift()
            let readStream = fs.createReadStream(readPath)
            readStream.on('end', () => {
                merge(paths)
            })
            readStream.on('error', (err) => {
                resolve([err])
                writeStream.close();
            });
            readStream.pipe(writeStream, { end: false })
            return
        }

        merge(filePaths)
    })
}

```
3. 在前端处理分片的方法中，我直接在方法体内部把索引写死成0了。做到续传的时候发现，索引必须是方法参数，第一次上传是0，续传就要按照后端给的索引传入了

4. 在写前端给后端发请求，传输分片的方法时，直接用Promise.all处理响应结果，一下子循环产生了上千个请求连接池。浏览器进程崩溃了。。。。。所以又单独写了一个控制并发请求的方法。经过测试，几千个分片都没事，文件合并用管道流也无压力。好开心！
```
function requestUnderMax(formDatas, requestFn, max) {
    return new Promise((resolve, reject) => {
        let i = 0;
        let isRejected = false
        let queue = [];
        let resArr = Array(formDatas.length);
        let error = false;
        function loop(formDatas) {
            if (error) return;
            if (formDatas.length === 0 ) {
                Promise.all(queue).then(() => {
                    resolve(resArr);
                })
                return
            }
            if (queue.length < max && !error) {
                let p = requestFn(formDatas.shift())
                .then(res => {
                    resArr[p.index] = res;
                    if (formDatas.length > 0) {
                        queue.splice(queue.indexOf(p), 1)
                        loop(formDatas)
                    }
                })
                .catch((err) => {
                    error = true
                    reject(err)
                })
                p.index = i;
                i++;
                queue.push(p)
                return loop(formDatas)
            }
            return
        }
        loop([...formDatas]);
    })
}   
```

#### 思考感悟
1. 看网上讲的原理是一回事，自己上手做是另外一回事；
2. 真不能过分依赖第三方模块，用着一时爽，不深究原理，就没收获；
3. 写代码前的准备工作太重要了，设计是相当重要的环节；

#### 结尾

源码地址：https://github.com/zhm19901224/easy-uploader

码字不易，该文章和源码都是本人原创。转载请注明出处，谢谢！
