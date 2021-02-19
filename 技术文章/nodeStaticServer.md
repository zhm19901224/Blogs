#### 实现思路
1. 脚本读取用户请求的url，拼接脚本执行的地址，作为需要在服务器查找资源的路径
2. 如果目录不存在返回404
3. 如果文件是目录类型，则返回目录链接列表，需要拼装当前文件夹下所有文件的链接。
4. 如果是文本类型文件，将内容以文本形式返回给用户。返回过程中，要通过对文件类型mine的匹配获取到文件类型，并且正确设置response header，使浏览器正确的解析。
5. 实现GZIP压缩，根据request header中，Accept-Encoding字段浏览器所支持的类型，设置response head中的Content-Encoding类型，并将压缩好的文本文件吐给前端
6. 缓存的实现。有默认的缓存参数，或者通过读取参数，配置协商缓存的具体信息。

#### 注意事项
1. 提供cli命令行工具
2. 代码中要处理好回调地狱
3. 保证可扩展性，配置文件要进行抽离

#### 部分请求处理代码
```
const { betterPromisify, fsStat, fsReadDir, errHandler, getContentType } = require('./utils');
const path = require('path');
const fs = require('fs')
const Handlebars = require("handlebars");
const compress = require('./compress')
const isNeedFresh = require('./cache')
const tplPath = path.join(__dirname, './template.tpl')
const tplHtml = fs.readFileSync(tplPath, 'utf8')

module.exports = async (req, res, config) => {
    // 访问的资源路径等于=脚本执行位置拼接用户访问的路径
    const filePath = path.join(config.root, req.url);
    const [statsErr, stats] = await fsStat(filePath);

    if (statsErr) {
        res.setHeader('Content-Type', 'text/plain')
        res.statusCode = 404;
        res.end(filePath+'is not directory or file, resource not found!')
        return
    }

    if (stats.isFile()) {
        const contentType = getContentType(filePath)
        res.setHeader('Content-Type', contentType)
        if (isNeedFresh(stats, req, res)) {
            res.statusCode = 304;
            res.end()
            return
        }

        res.statusCode = 200;
        let readStream = fs.createReadStream(filePath)
        if (filePath.match(config.comp)){
            // 流入压缩器
            readStream = compress(readStream, req, res)
        }
        
        readStream.pipe(res)
        
    } else if (stats.isDirectory()) {
        let [err, files] = await fsReadDir(filePath)
        if (err) return errHandler(err)
        res.setHeader('Content-Type', 'text/html')
        res.statusCode = 200;
        const html = Handlebars.compile(String(tplHtml))({ 
            dir: path.relative(config.root, filePath), 
            files: files.map(file => ({ file, type: getContentType(file) || 'dir' })),
            title: path.basename(filePath)
        })
        res.end(html)
    }
}
```

> 详情见 https://github.com/zhm19901224/hm-server