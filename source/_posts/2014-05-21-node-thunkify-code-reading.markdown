---
layout: post
title: "node-thunkify Code Reading"
date: 2014-05-21 12:02
comments: true
categories: codeReading 
---

##[thunkify](https://github.com/visionmedia/node-thunkify)



##code


```
function thunkify(fn) {
    assert('function' == typeof fn, 'function required');    

    return function () {
        var args = slice.call(arguments);    
        var ctx = this;
        return function (done) {   // done 是回调函数
            var called;
            args.push(function() {
                if (called) return;
                called = true;    
                done.apply(null, arguments); 
            }); // 将回调处理加入参数列表

            try {
                fn.apply(ctx, args);   // 函数处理 
            } catch (err) {
                done(err);   // 异常获取并用回调处理 
            }
        }
    }
};
```


##例子

```
var thunkify = require('thunkify');
var fs = require('fs');


var read = thunkify(fs.readFile);

read('package.json', 'utf8')(function(err, str){
});
```

