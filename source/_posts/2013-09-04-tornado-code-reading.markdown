---
layout: post
title: "Tornado code Reading"
date: 2013-09-04 13:26
comments: true
categories: codeReading
---

##服务器建立

[HTTPServer](https://github.com/facebook/tornado/blob/master/tornado/httpserver.py)

**应用**

```
application = web.Application([
(r"/", MainPageHandler),	
])
http_server = httpserver.HTTPServer(application)
http_server.listen(8080)
ioloop.IOLoop.instance().start()
```

对照应用例子理解源码

```
def __init__(self, request_callback, no_keep_alive=False, io_loop=None,
                 xheaders=False, ssl_options=None, protocol=None, **kwargs):
        self.request_callback = request_callback
        self.no_keep_alive = no_keep_alive
        self.xheaders = xheaders
        self.protocol = protocol
        TCPServer.__init__(self, io_loop=io_loop, ssl_options=ssl_options,
                           **kwargs) #HTTPServer 继承自 TCPServer, 初始化TCPServer
```

**参数**

- request_callback: 必须产生一个http回复, 例子中 application 即是回复
- xheaders: True(支持通过`x-real-ip`或`x-forwarded-for`获取ip) False(当torando之前有反向代理或者负载均衡self.request.remote_ip只能获得127.0.0.1)
- ssl_options: ssl传输数据

ssl_options 使用例子 
```
HTTPServer(applicaton, ssl_options={
   "certfile": os.path.join(data_dir, "mydomain.crt"),
   "keyfile": os.path.join(data_dir, "mydomain.key"),
	}) 
```
下面应该说下 `TCPServer` 主体内容在 `TCPServer`, 


[TCPServer](https://github.com/facebook/tornado/blob/master/tornado/tcpserver.py)


