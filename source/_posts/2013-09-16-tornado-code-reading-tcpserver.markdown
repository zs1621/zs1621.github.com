---
layout: post
title: "Tornado Code Reading-TCPServer"
date: 2013-09-16 16:31
comments: true
categories: codeReading
---

##[TCPserver](https://github.com/facebook/tornado/blob/master/tornado/tcpserver.py)


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

- `listen`: 监听单个进程

```
server = TCPServer()
server.listen(8888)
IOLoop.instance().start()
```

- `bind` / `start`: simple multi-process::

```
server = TCPServer()
server.bind(8888)
server.start(0) #Forks multiple sub-process
IOLoop.instance().start()
```

如果使用`start`接口， 一个 `.IOLoop` 不可以传入 `TCPServer` 结构里。 `start`将总是开始服务在默认的单例 `.IOLoop`

- `add_sockets`: 高级多进程::

```
sockets = bind_sockets(8888)
tornaado.process.fork_processes(0)
server = TCPServer()
server.add_sockets(sockets)
IOLoop.instance().start()
```

`add_sockets` 接口更为复杂， 但是和`tornado.process.fork_processes`使用将会更明了， `add_sockets`也可以用在单进程服务中 ， 如果你想要创建你的监听套节字而不是`~tornado.netutil.bind_sockets`

---

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

源码里的 `bind_sockets` 见 [bind_sockets](#bind_sockets), `add_sockets` 见下


**add_sockets**

让服务接受多个连接

`sockets` 参数是一个socket数组, `add_sockets`一般和 `tornado.process.fork_processes`联合使用 为了控制 多进程服务的启动

```
if self.io_loop is None:
	self.io_loop = IOLoop.current()

for sock in sockets:
	self._sockets[sock.fileno()] = sock
	add_accept_handler(sock, self._handle_connection, io_loop=self.io.loop) 
```

上面的 add_accept_handler 见 [add_accept_handler](#add_accept_handler)
add_accept_handler 第二个参数 self._handle_connection是个回调函数, 分析如下

> _handle_connection在接受客户端的连接处理结束后会被调用，调用时传入连接和ioloop对象初始化 IOStream,用于对客户端的异步读写;然后调用 `handle_stream`(注意这里的handle_stream 文档说了如果你只是用`tcpserver`那么，handle_stream得自己重写, 如果用tornado的httpserver 那handle_stream 在 [httpserver](https://github.com/facebook/tornado/blob/master/tornado/httpserver.py)), 传入创建的`IOStream`对象初始化一个`HTTPConnection`, `HTTPConnection`  封装了`IOStream` 的一些操作， 用于处俩 `HTTPRequest` 并返回。 至此 `HTTPServer`的创建、启动、注册回调函数过程结束


```python
#如果self.ssl_options不为空，处理ssl代码略
try:
	if self.ssl_options in not None:
		stream = SSLTOSTREAM(connection, io_loop=self.io_loop, max_buffer_size=self.max_buffer_size)
	else:
		stream = IOSTream(connection, io_loop=self.io_loop, max_buffer_size=self.max_buffer_size)
	self.handle_stream(stream, address)
except Excetpion:
	app_log.error('Error in connection callback', exc_info=True)
```

从上面的分析和源码 可知 服务器的工作流程 socket->bind->listen创建 listen socket 监听客户端， 并将每个listen socket 的 fd 注册到IOLoop的单例实例中; 当 listen socket 可读时回调 _handle_events 处理客户端请求；在与客户端通信的过程中使用 IOStream 封装读写缓冲区， 实现与客户端的异步读写。 下面我们将具体了解listen socket 的 fd 被注册到IOLoop的单例实例中  见 [IOLoop](http://zs1621.github.io/blog/2013/09/19/tornado-code-reading-ioloop/)

---

##[netutil](https://github.com/facebook/tornado/blob/master/tornado/netutil.py)

<a name="bind_sockets" id="bind_sockets">**bind_sockets**</a>
  - 解释:创建监听套节字绑定到给定的端口和地址

  - 参数
    - address: IP地址或主机名。如果是主机名，服务将会监听所有跟此域名有关的`IP`
    - Family: `socket.AF_INET` 或 `socket.AF_INET6`
    - backlog: 这个参数跟`socket.listen()<socket.socket.listen>`一样
    - flags: 

  - 代码

```python
sockets = []
if address == "":
	address = None
if not socket.has_ipv6 and family == socket.AF_UNSPEC:
	family = socket.AF_INET
if flags is None:
	flags = socket.AI_PASSIVE
	#见 [getaddrinfo](https://github.com/zs1621/pythostudy/wiki/socket)
	#`getaddrinfo`会返回服务器的所有网卡信息， 每个网卡都要监听客户端的请求并返回创建的sockets
for res in set(socket.getaddrinfo(address, port, family, socket.SOCK.STREAM, 0, flags)):
	af, socktype, proto, canoname, sockaddr = res
	try:
		#创建套节字
		sock = socket.socket(af, socktype, proto)
	except socket.error as e:
		if e.args[0] == errno.EAFNOSUPPORT:
			continue
		raise
	set_close_exec(sock.fileno())
	if os.name != 'nt':
		#TCP连接中，recv等函数默认为阻塞模式(block)，
		#即直到有数据到来之前函数不会返回，
		#而我们有时则需要一种超时机制使其在一定时间后返回而不管是否有数据到来，
		#这里我们就会用到setsockopt()函数 见 [setsockopt](http://blog.chinaunix.net/uid-25749806-id-348637.html)
		sock.setsockopt(socket.SOL_SOCKET, socket.SOREUSERADDR, 1)
	#在linux ipv6 sockets 也接受 ipv4，这样的话不能同时绑定 ipv4 和 ipv6.为了方便，在ipv6 sockets中总是禁用ipv4 
	if af == socket.AF_INET6:
		if hasattr(socket, "IPPROTO_IPV6"):
			sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY)
	sock.setblocking(0)
	sock.bind(sockaddr)
	#默认设定等待被处理的连接最大个数
	sock.listen(backlog)
	sockets.append(sock)
return sockets
```

<a name="add_accept_handler" id="add_accept_handler">**add_accept_handler**</a>: 添加一个`IOLoop`事件去接受新的连接在 `sock`

> 当一个连接被接受了， `callback(connection, address)`(`connection`是socket对象， `address`是连接的另外结尾处的地址)将会运行


```python
if io_loop is None:
 	io_loop = IOLoop.current()
def accept_handler(fd, events):
	while True:
		try:
			connection, address = sock.accept()
		except socket.error as e:
			#EWOULDBLOCK 和 EAGAIN 表明我们
			#已经接受了每个可以接受的连接
			#具体见[EWOULDBLOCK](http://stackoverflow.com/questions/3647539/socket-error-errno-ewouldblock)
			if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
				return
			#ECONNABORTED 表明有个连接还在接受队列时被关闭了
			if e.args[0] == errno.ECONNABORTED:
				continue
			raise
		callback(connection, address)
io_loop.add_handler(sock.fileno(), accept_handler, IOLoop.READ)
```

