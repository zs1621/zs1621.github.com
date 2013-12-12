---
layout: post
title: "Tornado Code Reading - tornado.concurrent"
date: 2013-11-08 13:38
comments: true
categories: 
---

##Future背景
Future 代表一个函数的调用结果. 函数返回一个值或者抛出一个异常， 所以future 包含一个值或者一个异常(可以通过future.result()获得这个结果，异常和返回值). Fuures存在与对应的函数结束前. 在 一个  多线程场景下,简单的调用 future.result()等待另外一个线程或者进程完成。在 异步场景下，你可以附带一个callback给future为了当调用结束可以得到通知。（with future.add_done_callback or io_loop.add_future） 

##了解 Future
`Futures` 已经在Python3.2应用了 [concurrent.futures](http://python.readthedocs.org/en/latest/library/concurrent.futures.html#concurrent.futures) , 如果在python3.2之前版本用 可以 (pip install futures). 那在 tornado 中如果可以将用就用python包，否则将会使用一个兼容的类 `tornado.concurrent.Future` 


## class tornado.concurrent.Future
这个类封装了异步操作的接口。在同步程序中， Futures被用来等待一个线程或者进程池的结果。在Tornado一般用在 IOLoop.add_future 或者 在一个 gen.coroutine 


```
"""
如果有 concurrent.futures 可用，那就用它; 否则用 _DummyFuture 
"""
if futures is None:
    Future = _DummyFuture
else:
    Future = futures.Future
```


## _DummyFuture

 - result

```
"""
先检查有没有完成 ，没有完成 ,直接抛出异常!如果有异常就raise 异常，否则返回 result
"""
def result(self, timeout=None):
	self._check_done()
	if self._exception:
		raise self._exception
	return self._result
```
 - exception

这个函数先与result唯一的不同是 无异常的时候 return None ;

 - add_done_callback(fn)

```python
"""
给Future加入一个回调。当它完成将会 以Future为参数调用回调函数. 
"""
def add_done_callback(self, fn):
	if self.done:
		fn(self)
	else:
		self._callbacks.append(fn)
```

## TracebackFuture
存储 异常的追踪


## DummyExecutor
```
class DummyExecutor(object):
	def submit(self, fn, *args, **kwargs):
		future = TracebackFuture()
		try:
			future.set_result(fn(*args, **kwargs))
		except Exception:
			future.set_exc_info(sys.exc_info())
		return future
	def shutdown(self, wait=True):
		pass
```

## run_on_executor

```python
def run_on_executor(fn):
    """
    异步的跑同步方法
    """
    @functools.wraps(fn):
    def wrapper(self, *args, **kwargs):
        callback = kwargs.pop("callback", None)
        future = self.executor.submit(fn, self, *args, **kwargs) #self.executor  理解为线程池
        if callback:
            self.io_loop.add_future(future, \
                lambda future: callback(future.result()))
        return future
    return wrapper
```

这里有关于`run_on_executor`的应用例子 https://gist.github.com/zs1621/7921770


TBC 
