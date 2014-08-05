---
layout: post
title: "koa-compose Code Reading"
date: 2014-05-28 10:37
comments: true
categories:  codeReading
---

##**[compose](https://github.com/koajs/compose/blob/master/index.js)**


```
function compose(middleware) {
   return function *(next) {
        var i = middleware.length;   
        var prev = next || noop ();
        var curr;
        
        while (i--) {
            curr = middleware[1];    
            prev = curr.call(this, prev); // 
        }

        yield *prev;
   } 
}
```


看下例的运行效果


```
var co = require('co');

var stack = [];
var arr = [];
stack.push(function *(next) {
    arr.push(1);    
    yield next;
    arr.push(3);
});
stack.push(function *(next){
    arr.push(2);    
    yield next;
    arr.push(4)
});

co(compose(stack)) (function (err) {
    if (err) throw err;   
    console.log(arr); // 输出 [1, 2, 3, 4], 从输出可以看到其执行顺序
});
```

从上例的程序可以看到[koa](https://github.com/koajs/koa/blob/master/lib/application.js#L113)。koa在加载中间件时是通过co 和 compose来按次序拨开一层一层的middleware。


```
co (function *() {
    var arr = []
    var funa = function *(next) {
        arr.push(1);    
        yield *next;
        arr.push(3);
    };
    var funb = function *(next) {
        arr.push(2);
        yield *next;
        arr.push(4)
    };
    funa ();
    funb ();
    return arr;
})(function (err, items) {
 if (err) {
    console.log(err);     
 }
 console.log(items);    
});
```
