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
    /*
     * 主要需要監測socket和res的事件 
     */
    var socket = thingie.socket || thingie;    
    var res = thingie.res || thingie; 

    /*
     * res完成或者socket不可寫的時候， 直接執行callback, 返回thingie
     */
    if ( res.finished || !socket.writable) {
        defer(callback);    
        return thingie;
    };  

    /*
     * 第一次運行時listener 和 res.__onFinished 是不存在的
     */
    var listener = res.__onFinished;

    if (!listener || !listener.queue) {
        listener = res.__onFinished = function onFinished(err)  {
            if (res.__onFinished === listener) res.__onFinished = null; // 這句表明每次給listener 賦完值， res.__onFinished = null;
            var queue = listener.queue || [];
            while (queue.length) queue.shift()(err)
        }   
        listerner.queue = [];

        first([
            [socket, 'error', 'close'], 
            [res, 'finish'],
        ], listener); // 看[first](https://github.com/jonathanong/ee-first)模块解析: 主要功能在监听到指定的事件的第一件时，触发响应回调， 并清除移除所有的监听。
    }

    listener.queue.push(callback);

    return thingie;
}
```


##*想要具体看finish的应用场景，其实在[单元测试](https://github.com/expressjs/finished/blob/master/test/test.js)里有很多具体的例子*
 - 下面具体理解下其中的一个单元测试



```
 describe('when the request finished', function () {
    it('should execute the callback when called after finish', function(done){
        var server = http.createServer(function(req, res){
            onFinished(res, function(){
                //第4步
                onFinished(res, done); // 第5步   
            }) // 第2步   
            setTimeout(res.end.bind(res), 0); // 第3步
        })  // 第 0 步  
        server.listen(function(){
            var port = this.address().port;    
            http.get('http://127.0.0.1:' + port, function(){
                res.resume();
                res.on('close', server.close.bind(server));
            }) //第1步
        })
    })    
 })
```


- **测试步骤**
 - 先创建服务器 第0步
 - 监听端口,做出请求 第一步
 - 服务器收到请求， 执行回调函数 `onFinished(res, function(){})` 第2步
 - 第3步的话  结束请求 res.end.bind(res)
 - 第4步  监听到结束, 执行回调 `onFinished(res, done)`
 - 第5步  完成done



- **测试结论**:当服务器完成请求后，有onFinished()的话， 直接执行回调, 然后结束

