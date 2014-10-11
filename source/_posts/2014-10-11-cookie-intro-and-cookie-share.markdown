---
layout: post
title: "cookie intro and cookie share"
date: 2014-10-11 10:24:06 +0800
comments: true
categories: web basic
---

##cookie intro

 - ###how cookie come 
   - when you first visit xxx.com, the website produce a cookie and sent you on `http response header`  you may find the field `Set-Cookie`. 
 - ###why cookie
   - make website recongize client
   - website according the cookie know you and give you different website or advertisement. or remeber your login status ...
 - ###cookie attribuate
 

```
Set-Cookie:BAIDUID=17EFBEC8FFD5F79747775326540CB66C:FG=1; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com
```

   - HttpOnly
    - this atribute directs browsers can not expose the cookie through channels other http(s), so if httponly ,`alert(document.cookie)` should output null. 
   - secure
    - It's also a flag, it meant browsers transfer cookie through secure methods even if it's not https request. more detail at [here](http://stackoverflow.com/questions/2321224/cookie-across-http-and-https-in-php)
   - domain
    - domain value for cookie share, if a.xxx.com and b.xxx.com need share the cookie, domain='.xxx.com' is ok. more detail [1](http://stackoverflow.com/questions/3089199/can-subdomain-example-com-set-a-cookie-that-can-be-read-by-example-com?lq=1) 
   - path
    - path easy to understand, cookie is effective in which path
   - expires
    - the time the cookie will be deleted
   - max-age
    - the seconds cookie get delete,   the differences between `max-age` and `expires`, can ref [difference](http://mrcoles.com/blog/cookies-max-age-vs-expires/) 
   - name=value
    - easy to understand, you want to save what value in cookie, the example `BAIDUID=****`
   
*ps the cookie is produced by server*
   