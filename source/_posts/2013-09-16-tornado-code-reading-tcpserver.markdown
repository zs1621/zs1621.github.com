---
layout: post
title: "Tornado Code Reading-TCPServer"
date: 2013-09-16 16:31
comments: true
categories: 
---

[TCPserver](https://github.com/facebook/tornado/blob/master/tornado/tcpserver.py)


一个非阻塞， 单进程的 `TCP` 服务器

为了使用 `TCPServer`, 定义一个子类改写其中的 `handle_stream` 方法

如果让服务器通过 `SSL` 传输， 如下例

```
TCPServer(ssl_options={
	"certfile": os.path.join(data_dir, "mydomain.crt"),
	"keyfile": os.path.join(data_dir, "mydomain.key"),
})
```

`TCPServer` simple single-process::

1. `listen`: 监听单个进程

```
server = TCPServer()
server.listen(8888)
IOLoop.instance().start()
```

2. `bind` / `start`: simple multi-process::

```
server = TCPServer()
server.bind(8888)
server.start(0) #Forks multiple sub-process
IOLoop.instance().start()
```

如果使用`start`接口， 一个 `.IOLoop` 不可以传入 `TCPServer` 结构里。 `start`将总是开始服务在默认的单例 `.IOLoop`

3. `add_sockets`: 高级多进程::

```
sockets = bind_sockets(8888)
tornaado.process.fork_processes(0)
server = TCPServer()
server.add_sockets(sockets)
IOLoop.instance().start()
```

`add_sockets` 接口更为复杂， 但是和`tornado.process.fork_processes`使用将会更明了， `add_sockets`也可以用在单进程服务中 ， 如果你想要创建你的监听套节字而不是`~tornado.netutil.bind_sockets`


以上是怎么应用
下面看源码

**初始化**
```
def __init__(self, io_loop=None, ssl_options=None, max_buffer_size=None):
	self.io_loop = io_loop
	self.ssl_options = ssl_options
	self.sockets = {}
	self._pending_sockets = []
	self._started = False
	self.max_buffer_size = max_buffer_size
	#下面是检验 SSL 文件，此处略
```


**listen**

在给定的端口接受连接
这个方法可能被调用多次为了监听多个端口。`listen` 立即生效; 之后不必调用`TCPServer.start` , 但是， `,IOLoop` 是必要的

```
sockets = bind_sockets(port, address=address)
self.add_sockets(sockets)
```

源码里的 `bind_sockets` 见 [netutil]("#netutil"), `add_sockets` 见下


**add_sockets**

让服务接受多个连接

`sockets` 参数是一个socket数组, `add_sockets`一般和 `tornado.process.fork_processes`联合使用 为了控制 多进程服务的启动

```
if self.io_loop is None:
	self.io_loop = IOLoop.current()

for sock in sockets:
	self._sockets[sock.fileno()] = sock
	add_accept_handler(sock, self._handle_connection, io_loop=self.io.loop) #这里的 add_accept_handler 见 [netutil](https://github.com/facebook/tornado/blob/master/tornado/netutil.py)
```

```

```
