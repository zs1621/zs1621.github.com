---
layout: post
title: "connect code reading"
date: 2013-09-02 22:12
comments: true
categories: 
---

[lib -> patch.js](https://github.com/zs1621/connect/blob/master/lib/patch.js)

**headerSent**

```
res.__defineGetter__('headerSent', function(){
    return this._header;
  });
```

这里的this._header是header内容发出后才有的， 内容为header的所有内容， 所以headerSent 只有在res.end()后才有值



[lib->middle->compress.js](https://github.com/zs1621/connect/blob/master/lib/middleware/compress.js)

**options**

compress中间件用了nodejs 内部模块 `zlib`

这里的options 参考 [options](http://nodejs.org/api/zlib.html#zlib_options)

```
res.on('header', function(){

})
```

监听到 `header` 开始

 *  如果没有compress 直接return;
 *  `var encoding = res.getHeader('Content-Encoding') || 'identity' `
 *  如果 `encoding` 不等于 `identity` 说明已经编码了 直接 return  
 *  如果需要压缩的文件 符合过滤条件 也即在给的文件列表中 就继续下一步, 否则 return  
 *  如果请求头文件 没有 accept-encoding 值 ， return (why)
 *  如果请求方法 为 `HEAD` , `return`
 *  默认压缩方法为 `gzip`
 *  确定压缩方法 即给 `method` 赋值
 *  此时`method` 还是空的话  return 
 *  压缩流文件 `stream = exports.methods[method](options)` 即 `zlib.createGzip` 或 `zlib.createDeflate` 得到 流 `stream`
 * 开始赋值 `Content-Encoding` , 并去除 `Content-Length` 明显压缩后 文件大小会变
  此时 `stream` 监听 到 发出去的数据 就 开始 res.write`stream.write(new Buffer(chunk, encoding))` 这里的 stream 监听 是 `node version 0.10`后的`stream api` 具体 参考 [stream](https://github.com/joyent/node/blob/master/doc/api/stream.markdown)
 * 直到 流文件 `"流出"`


