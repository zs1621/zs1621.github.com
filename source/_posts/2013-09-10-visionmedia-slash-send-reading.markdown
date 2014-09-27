---
layout: post
title: "visionmedia/send Reading"
date: 2013-09-10 00:41
comments: true
categories: codeReading
---

**`send`入口**

```
exports = module.exports = send;
```

**`send` 函数返回 `SendStream` 构造函数**

```
function send(req, path, options) {
	return new SendStream(req, path, options);
}
```

**来看`SendStream`**

 - 参数 
   - req 即 `request`
   - path : `string` 路径
   - options: 

```
function SendStream(req, path, options) {
	....
}
```
 - 继承 `Stream.prototype`

```
SendStream.prototype.__proto__ = Stream.prototype;
```

 - 下面都是 `SendStream` 原型链的方法
   - hidden: 赋值 this._hidden, return this
   - index: 赋值默认的 index `path`, this._index, return this
   - root: 赋值根路径,  this._root 
   - maxage: 赋值最大缓存时间 , this._maxage
   - error: 根据`status`->触发`error`
   - redirect: 如果有监听事件`directory`的监听者 那么触发`directory`
   - isMalicious: 检测`pathname`是否有潜在的问题,判断方法如果 没有`_root`且 `path`包含`..` 那么 就是有异常的路径;
   - hasTrailingSlash: 判断路径最后一位是不是`/`
   - hasLeadingDot: 就是文件是否有后缀 形如`aa.html`
   - isCachable: 是否缓存， 判断方法 `(res.statusCode >= 200 && res.statusCode < 300) || 304 == res.statusCode`
   - isFresh: 判断缓存是否是新的 判断方法 引用模块 `fresh`
   - removeContentHeaderFields 去除包含 `content`的头 `key`
   - onStatError: 

 - 主体方法 `pipe` , 参数 `res`
```
var self = this
, args = arguments
, path = this.path
, root = this._root

this.res = res;
//判断uri的有效性
path = utils.decode(path);
if (-1 == path) return this.error(400);

//如果`path`里包含`\0` ，表明`path`为空
if (~path.indexOf('\0')) return this.error(400);

//连接 this._root 和 path
if (root) path = normalize(join(this._root, path));

//如果异常路径禁止访问
if (this.isMalicious()) return this.error(403);

//此时`path`已经是 `this._root`和 `path`的结合, 所以此种情况不会出现,如果有root
if (root && 0 != path.indexOf(root)) return this.error(403)

//如果_hidden为`false`,那么不支持隐藏文件，此时文件如果以`.`开头那么就不支持 
if (!this._hidden && this.hasLeadingDot()) return this.error(404);

//index 文件支持
if (this._index && this.hasTrailingSlash()) path += this._index

//最后看path 的信息
fs.stat(path, function(err, stat){
	if (err) return self.onStatError(err); //上面已介绍
	if (stat.isDirectory) return self.redirect(self.path); //见上
	self.emit('file', path, stat); //见下
	self.send(path, stat);//见下
});

```
  - 方法 **`send`**
    - path: 路径
    - stat: 文件状态
  
```
var options = this.options;
var len = stat.size;
var res = this.res;
var req = this.req;
var ranges = req.headers.range; //截取部分文件之用
var offset = options.start || 0;

//赋值header
this.setHeader(stat);

//赋值 content-type
this.type(path);

//条件 GET 支持, isFresh() 见 [fresh](https://github.com/visionmedia/node-fresh/blob/master/index.js)
if (this.isConditionGET()
&& this.isCachable()
&& this.isFresh()) {
	return this.notModified();
}

//
len = Math.max(0, len - offset);
if (options.end != undefined) {
	var bytes = options.end - offset + 1;
	if (len > bytes) len = bytes;
}

// Range 支持
if (ranges) {
	...	
}

// content-length
res.setHeader('Content-Length', len);

// HEAD support
if ('HEAD' == req.method) return res.end();

this.stream(path, options); //见下
```

  - 方法 `stream`
    - 参数
       - path : 路径
       - options

```
var self = this;
var res = this.res;
var req = this.req;

//pipe 把流数据加入 `res`管道
var stream = fs.createReadStream(path, options);
this.emit('stream', stream);
stream.pipe(res);

//socket 关闭， 

req.on('close', stream.destroy.bind(stream));

//error处理
steam.on('error', function(err){
	//不回复
	if (res._header) {
		console.error(err.stack);
		req.destroy();
		return;
	}
	erro.status = 500;
	self.emit('error', err);
});

//end
stream.on('end', function(){
	self.emit('end');	
});
```
		

