---
layout: post
title: "connect code reading"
date: 2013-09-02 22:12
comments: true
categories: 
---

[lib -> patch.js] (https://github.com/zs1621/connect/blob/master/lib/patch.js)

**headerSent**

```
res.__defineGetter__('headerSent', function(){
    return this._header;
  });
```

这里的this._header是header内容发出后才有的， 内容为header的所有内容， 所以headerSent 只有在res.end()后才有值


