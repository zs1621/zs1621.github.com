---
layout: post
title: "Tornado Code Reading - IOLoop"
date: 2013-09-19 12:36
comments: true
categories: codeReading
---

##[IOLoop](https://github.com/facebook/tornado/blob/master/tornado/ioloop.py)

对着IOLoop 的源码瞅了一天，楞是没有明白， 为什么 父类实例可以调用子类方法?

最后看到[configurable](http://hswg.info/blog/2013/03/24/configurable-of-tornado-note/)
[tornado 解读](http://ispe54.blogspot.com/2013/04/tornado-7.html)

看一下可配置接口的实现

<a name='IOLoop' id='IOLoop'>configurable两函数</a>
```python
@classmethod
def configurable_base(cls):
	return IOLoop

@classmethod
def configurable_default(cls):
	if hasattr(select, "epoll"):
		from tornado.platform.epoll import EPollIOLoop
		return EPollIOLoop
	if hasattr(select, "kqueue"):
		#python 2.6+ on BSD or Mac
		from tornado.platform.kqueue import KQueueIOLoop
		return KQueueIOLoop
	from tornado.platform.select import SelectIOLoop
	return SelectIOLoop
```

> Configurable类是可配置接口的父类， 可配置接口对外提供一致的接口标志， 但它的子类实现可以在运行时进行configure。一般跨平台时由于子类实现有多种选择， 这时候就可以使用配置接口， 例如 select 和 epoll。首先注意 Configurable 的两个函数: configurable_base 和 configurable_default, 两函数都需要被子类(即可配置接口类)覆盖重写。其中， base函数一般返回接口类自身， default 返回接口的默认子类实现， 除非接口指定了 __impl_class。IOLoop及其子类实现都没有实现初始化函数也没有构造函数， 七构造函数继承于 Configurable, 如下::


```
def __new__(cls, **kwargs):
	base = cls.configurable_base()
	args = {}
	if cls is base:
		impl = cls.configured_class()
		if base.__impl_kwargs:
			args.update(base.__impl_kwargs)
	else:
		impl = cls
	args.update(kwargs)
	instance = super(Configurable, cls).__new__(impl)
	instance.initialize(**args)
	return instance
```
当子类对象被构造时， 子类__new__被调用， 因此参数里的cls 指的是Configurable的子类(可配置接口类， 如IOLoop)。先得到base,  [IOLoop代码](#IOLoop) 可知, configurable_base返回的是自身类。由于 base 和 cls 是一样的， 所以调用 configured_class() 得到接口的子类实现(见[configured_class](#configured_class)) 其实就是调用 base的 configurable_default(?????TBD), 就是返回一个子类实现(epoll/kqueue/select之一),顺便把__impl_kwargs合并到args 里 。然后调用Configurable类的父类(Object)的 __new__方法， 生成一个impl的对象， 紧接着把args当参数调用该队想的initialize(继承PollIOLoop) , 返回该对象。 所以， 当构造IOLoop对象时， 实际得到的是EPollIOLoop或其它相关子类。可以看出， Configurable 类主要提供构造方法， 相当于对象工厂根据配置来生产对象， 同时开放configure接口以供配置。而子类按照约定调整配置即可得到不同对象， 代码得到了复用 或其它相关子类。可以看出， Configurable 类主要提供构造方法， 相当于对象工厂根据配置来生产对象， 同时开放configure接口以供配置。而子类按照约定调整配置即可得到不同对象， 代码得到了复用   

上面的过程如果不好太理解  可以去看  [example](https://github.com/zs1621/pythostudy/blob/master/tcp/tcp_loop_server.py) 这样大致能理解 ioloop 实例的初始化过程

<a name="configured_class" id="configured_class">configured_class</a>
```
base = cls.configurable_base()
if cls.__impl_class is None:
	base._impl_class = cls.configurable_default()
return base.__impl_class
```

===

上面主要解释了 IOLoop 为什么能调用子类方法  以及  可配置接口的实现
下面来看 IOLoop 的对象 instance

IOLoop 实现了单例的概念, 具体见 [IOLoop单例](https://github.com/zs1621/pythostudy/blob/master/class/singleton.py)


理解了上面的概念 接着 [TCPServer](http://zs1621.github.io/blog/2013/09/16/tornado-code-reading-tcpserver/) 最后的 add_handle !其实 此时的 object 已经是确定的 `EOLoop` 或者 `Kqueue` 的对象！ 这里的 add_handle 是 它们的父类 `POLoop` 的 方法， 这明显就是继承了! 

add_handler 代码如下， 首先把 处理方法的上下文 存入 _handlers ，等调用时再恢复， 这个机制是 statck_context 见 [statck_content]() 做到的。 第二步 先来看下  self._impl 从哪里来  -> self._impl = impl 此时需要知道是谁调用 initialize  -> 这里初始化是在构造函数 __new__ 里调用的, `instance.initialize(**args)`此时的instance 为 `EPollIOLoop` 实例 ->  `super(EPollIOLoop, self).initialize(impl=select.epoll, **kwargs)` -> EPollIOLoop 的父类的 initialize() 很明显 impl 为 select.epoll[epoll](http://docs.python.org/2/library/select.html)

```
 def add_handler(self, fd, handler, events):
 	self._handlers[fd] = statck_context.wrap(handler)
	self._impl.register(fd, events | self.ERROR)
```

这步呢 就是把 监听 fd 和 accept_handler方法进行关联, 至此事件分发到此就结束了


======

下面来看下 IOLoop 的主循环 `start()`

```
def start(self):
	if not logging.getLogger().handlers:
		logging.basicConfig
	if self._stopped:
		self._stopped = False
		return
	old_current = getattr(IOLoop._current, "instance", None)
	IOLoop._current.instance = self
	self._thread_ident = thread.get_ident() #Return the ‘thread identifier’ of the current thread. This is a nonzero integer
	self._running = True

	old_wakeup_fd = None
	if hasattr(singal, 'set_wakeup_fd') and os.name == 'posix':
		try:
			old_wakeup_fd = signal.set_wakeup_fd(self._waker.write_fileno())
			if old_wakeup_fd != -1:
				signal.set_walkeup_fd(old_wakeup_fd)
				old_walkup_fd = None
		except ValueError:
			pass

```

上面这段代码 TBD


```
while True:
	poll_timeout = _POLL_TIMEOUT
	#Prevent IO event starvation by delaying new callbacks 
	# to the next iteration of the event loop.
	with self._callback_lock:
		callbacks = self._callbacks
		self._callbacks = []
	for callback in callbacks:
		self._run_callback(callback)
	# Closures may be holding on to a lot of memory, so allow
	# them to be freed before we go into our poll wait.
	callbacks = callback = None

	if self._timeouts:
		now = self.time()
		while self_timeouts:
			if self._timesouts[0].callback is None:
				# the timeout was cancelled
				heapq.heappop(self._timeouts)
				self._cancellations -= 1
			elif self._timeouts[0].deadline <= now:
				timeout = heapq.heappop(self._timeouts)
				self._run_callback(timeout.callback)
				del timeout
			else:
				seconds = self._timeouts[0].deadline - now
				poll_timeout = min(seconds, poll_timeout)
				break

```
