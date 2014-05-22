---
layout: post
title: "Lazy.js code Reading"
date: 2013-12-18 08:58
comments: true
categories: codeReading
---

###[Lazy.js](https://github.com/dtao/lazy.js)


在看源码前， 可以先看下Lazy.js的基本思想 [Lazy.js 的设计模式](https://github.com/zs1621/lazy.js/wiki/%E4%BB%8B%E7%BB%8D-LAZYJS(%E7%BF%BB%E8%AF%91))


###Lazy 

```javascript
Lazy([1, 2, 4]) // in [1, 2, 4] -> out instanceof Lazy.ArrayLikeSequence 
Lazy({ a: 'b'})
Lazy("hello world")
Lazy()
Lazy(null)
```

 - 输入参数 Array | object | string |  | null
 - 输出 { source: [1, 2, 4] } | { source: {'a': b} } | { source: "hello world"} | { source: undefined }  | { source: null } 
 - 可以看到输入的参数被打包成相应的对象, 而这些对象都继承自 `sequence`

###Sequence


`sequence` 对象提供对 0或者更多连续元素的集合 的统一的 API 封装.为什么所有的操作需要一个 sequence. 看下面的例子

```javascript
var seq = Lazy(source) // 1st sequence
        .map(func) // 2nd MappedSequence
       .filter(pred) // 3rd FilteredSequence
        .reverse() // 4th ReversedSequence
seq.each(function(x) { console.log(x); })
```

上面这个例子中 前四步除了创建对应的sequece没有做任何的遍历source或者别的操作。 只有在第5步调用 each 时，将一次性按照鍊條(chain)的順序处理source 得到最后的结果。所以lazy做的就是延迟遍历处理数据.

> in fact, when i think about  the performance of `underscore` and `lazy.js`; i cann't understand why lazy is faster.  lazy.js: 1 2s 3 underscore: 1 2 3 1 2 3 1 2 3 . so what's the difference. lazy.js just hold off some process; i cann't get it.... so continue to read code. >_< 


```javascript
function Sequence() {} # 创建Sequence构造函数

```


TBC

