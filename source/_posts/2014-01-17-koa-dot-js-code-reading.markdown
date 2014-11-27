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

 - version： 0.8.2
 - 用到的库
   - escape-html: =1.0.1
   - statuses: =1.0.1
   - accepts: 1.0.0
   - type-is: 1.1.0
   - set-type: 1.0.0
   - mime-types: 1.0.0
   - [finished: 1.2.0](https://zs1621.github.io/blog/2014/07/14/expressjs-slash-finished-code-reading/)
   - [co: 3.0.2](https://zs1621.github.io/blog/2014/11/27/co-coding-reading/)
   - debug
   - fresh: 0.2.1
   - [koa-compose: 2.2.0](https://zs1621.github.io/blog/2014/05/28/koa-compose-code-reading/)
   - koa-is-json: 1.0.0
   - cookies: 0.4.0
   - delegates: 0.0.3
   - [delegates](https://github.com/visionmedia/node-delegates/blob/master/index.js): 0.0.3 -已读 -13/5
   - dethroy: 1.0.0
   - vary: 0.1.0
   - error-inject: 1.0.0
   - parseurl: 1.2.0
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


来到`app.callback`


```
app.callback = function () {
    var mw = [respond].concat(this.middleware);   // 实际上 respond 就是最后一个需要执行的中间件， 负责返回数据的 
    var gen = compose(mw); 
    var fn = co(gen);  // co 和 compose 就是将一个个middle，按序执行的操作, 此时middle还没有运行
    var self = this;
    
    if (!this.listeners('error').length) this.on('error', this.onerror);
    
    return function (req, res) {
        res.statusCode = 404;    
        var ctx = self.createContext(req, res);
        finished(ctx, ctx.onerror);
        fn.call(ctx, ctx.onerror); // 这步运行了说明服务器接收到了请求， 然后一步步的运行middle 和 请求业务处理
    }
}
```


上面的代码基本上就解释了整个请求到回复的流程


附张图, 一图胜千言

![gif](https://raw.githubusercontent.com/koajs/koa/master/docs/middleware.gif)



下面分解开来

 - `createContext(req, res)`
 - `finished(ctx, ctx.onerror)`
 - `fn.call(ctx, ctx.onerror)`



###app.createContext(req, res)
 - 参数 req,res
 - 返回内容:context, context具体内容如下
  - context: `Object.create(this.context)`
  - context.app: `request.app=response.app=`
  - context.req: `request.req = response.app = req`
  - context.res: `request.res = response.res = res`
  - context.ctx: `response.ctx = context`
  - context.onerror: `context.onerror.bind(context)` [this.context](https://github.com/koajs/koa/blob/master/lib/context.js#l98)
  - context.originalUrl: `request.originalUrl = req.url`
  - context.cookie: `new Cookies(req, res, this.keys)` -> [cookie](https://github.com/expressjs/cookies) TODO
  - context.accept: `request.accept = accepts(app)` -> [accepts](https://github.com/jshttp/accepts)



###finished(ctx, ctx.onerror)


TBC
