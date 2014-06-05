---
layout: post
title: "Koa.js Code Reading"
date: 2014-01-17 15:23
comments: true
categories: codeReading
---
TBC 继续挖坑待填...

##[Koa](https://github.com/koajs/koa)

 
源码简况

 - version： 0.6.0
 - 用到的库
   - escape-html: =1.0.1
   - statuses: =1.0.1
   - accepts: 1.0.0
   - type-is: 1.1.0
   - set-type: 1.0.0
   - finished: 1.1.1
   - co: 3.0.2
   - debug
   - fresh: 0.2.1
   - koa-compose: 2.2.0
   - koa-is-json: 1.0.0
   - cookies: 0.4.0
   - [delegates](https://github.com/visionmedia/node-delegates/blob/master/index.js): 0.0.3 -已读 -13/5
   - dethroy: 1.0.0
   - error-inject: 1.0.0
   - [only](https://github.com/visionmedia/node-only): 0.0.2  -已读 -9/5
 - 运行条件: node 版本大于0.11.9



##**从入口说起**


```
var koa = require('koa');
var app = koa();

app.use(function *(){
    this.body = 'Hello World';    
});

app.listen(3000);
```


上面是一个最简单的服务,输出`Hello World`; 我们用到两个koa的api

 - app.use()
 - app.listen();

下面就先从app.listen() 开始


看 [application.js](https://github.com/koajs/koa/blob/master/lib/application.js) 


```
app.listen = function () {
    var server = http.createServer(this.callback()) ;  
    return server.listen.apply(server, arguments);
}
```

> http.createServer([requestListener])
 returns a new web server object

> `requestListener` is a function which is automatically added to the `'request'` 事件


由代码可以知道 this.callback() 就是一个 `requestListener`;


TBC





