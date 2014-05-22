---
layout: post
title: "connect code reading"
date: 2013-09-02 22:12
comments: true
categories: codeReading
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



[lib -> middle -> basicAuth.js](https://github.com/senchalabs/connect/blob/master/lib/middleware/basicAuth.js)

**3种应用形式**

 - 简单输入 username , password 
```
  connect(connect.basicAuth('username', 'password'));
```

 - 回调

```
   connect()
   .use(connect.basicAuth(function(user, pass){
      return 'tj' == user & 'wahoo' == pass;
   }))
```
 - 异步回调

```
   connect()
   .use(connect.basicAuth(function(user, pass, fn){
  	User.authenticate({user: user, pass: pass}, fn); 
   }))
```

 
**Authrization**

页面需要授权的时候，浏览器会弹出一个登入窗口, 输入正确的帐号， 浏览器会发送一个ＨＴＴＰ请求， 但此时会包含这样的一个 Header:

```
Authorization: Basic bXl1c2VyOm15cGFzcw==
```

这里的 `bXl1c2VyOm15cGFzcw==` 是 `base64 encoded`。例如, `base64_decode('bXl1c2VyOm15cGFzcw==')` -> `myuser:mypass`

理解头文件的 Authrization, 下面来看代码

```
      if ('string' == typeof callback)  //对应第一个应用例子, 见上 
      callback = function(user, pass) {
     		return user == username && pass == password; 
      } //这时给callback赋值 function
```

下面解析出头文件的即 `req.headers.authorization` 解码出用户输入的用户名和密码得到 `user` 和 `pass`

最后调用`callback`做验证 分一下两种情况

  - 异步callback
```
if (callback.length >= 3) {
	var pause = utils.pause(req); 
	callback(user, pass, function(err, user){
		if (err || !user) return unauthorized(res, realm);	
		req.user = req.remoteUser = user;
		next();
		pause.resume(); // nodejs version < 0.10, 在中间间使用异步调用时使用， 为了在异步操作完成后， 重新触发消息， 以免事件丢失
	});
}
```

  - 同步callback
```
 if (callback(user, pass){
	req.user = req.remoteUser = user; 
	next();
 }else {
	unauthorized(res, realm); 
 })
```

[lib ->middle ->static.js](https://github.com/zs1621/connect/blob/master/lib/middle/static.js)

**应用**

```
connect()
	.use(connect.static(__dirname+ '/public', {maxAge: 1000}))
```


**参数**

`root`: 

`options`:
	- 'maxAge' 浏览器缓存时间  默认为0
	- 'hidden' 允许隐藏文件的转换 默认false
	- 'redirect' 如果路径是一个文件夹， 指向 ‘/’ 默认 true
	- 'index' 默认的文件名‘index.html’

代码主体就是

```
send(req, path)
	.maxage(options.maxAge || 0)
	.root(root)
	.index(options.index || 'index.html')
	.hidden(options.hidden)
	.on('error', error)
	.on('directory', directory)
	.pipe(res)
```

对于 `send` 模块分析见 [send](zs1621.github.io/blog/2013/09/10/visionmedia-slash-send-reading)




