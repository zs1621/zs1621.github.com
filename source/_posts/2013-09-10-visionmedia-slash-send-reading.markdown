---
layout: post
title: "visionmedia/send Reading"
date: 2013-09-10 00:41
comments: true
categories: 
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
		

##TBD :)
