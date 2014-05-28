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


看下运行效果


```
var co = require('co');

var stack = [];
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

