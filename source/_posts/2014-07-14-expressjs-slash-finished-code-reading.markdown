---
layout: post
title: "expressjs/finished Code Reading"
date: 2014-07-14 11:57:26 +0800
comments: true
categories: codeReading 
---


##* [finished](https://github.com/expressjs/finished) 
 
 - 功能: 在请求关闭，完成， 或者出错时执行回调
 - 用例: 清除传输流。例如为了防止文件描述符泄漏，在socket出错时， 你会想要将文件流摧毁


##**从一个koa例子开始**

```
var onFinished = require('finished');
function* () {
    var stream = this.body = fs.createReadStream('thingie.json');   
    onFinished(this, function(err) {
        stream.destroy();    
    })
}
```



再看finished的代码


```
module.exports = function finished (thingie, callback) {
    var socket = thingie.socket || thingie;    
    var res = thingie.res || thingie;  // 主要需要监测socket, res的事件

    if ( res.finished || !socket.writable) {
        defer(callback);    
        return thingie;
    };   // res完成或者socket不可写的时候， 直接执行callback, 返回thingie

    var listener = res.__onFinished; 

    if (!listener || !listener.queue) {
        listener = res.__onFinished = function onFinished(err)  {
            if (res.__onFinished === listener) res.__onFinished = null;
            var queue = listener.queue || [];
            while (queue.length) queue.shift()(err)
        }   
        listerner.queue = [];

        first([
            [socket, 'error', 'close'], 
            [res, 'finish'],
        ], listener); // 看下面的first模块解析: 主要功能在监听到指定的事件的第一件时，触发响应回调， 并清除移除所有的监听。
    }

    listener.queue.push(callback);

    return thingie;
}
```

