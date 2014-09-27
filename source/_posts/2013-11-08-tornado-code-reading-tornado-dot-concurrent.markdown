---
layout: post
title: "Tornado Code Reading - tornado.concurrent"
date: 2013-11-08 13:38
comments: true
categories: codeReading
---

##Future背景
Future 代表一个函数的调用结果. 函数返回一个值或者抛出一个异常， 所以future 包含一个值或者一个异常(可以通过future.result()获得这个结果，异常和返回值). Fuures存在与对应的函数结束前. 在 一个  多线程场景下,简单的调用 future.result()等待另外一个线程或者进程完成。在 异步场景下，你可以附带一个callback给future为了当调用结束可以得到通知。（with future.add_done_callback or io_loop.add_future） 

##了解 Future
`Futures` 已经在Python3.2应用了 [concurrent.futures](http://python.readthedocs.org/en/latest/library/concurrent.futures.html#concurrent.futures) , 如果在python3.2之前版本用 可以 (pip install futures). 那在 tornado 中如果可以将用就用python包，否则将会使用一个兼容的类 `tornado.concurrent.Future` 


## class tornado.concurrent.Future
**这个类封装了异步操作的结果**。在同步程序中， Futures被用来等待一个线程或者进程池的结果。在Tornado一般用在 IOLoop.add_future 或者 在一个 gen.coroutine 


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

## return_future
 - what use: 让函数通过回调返回一个`Future`
 - how use: @return_future

> 看 tornado 源码的 test文件 concurrent_test.py; 


```python
class ReturnFutureTest(AsyncTestCase):
    @return_future
    def sync_future(self, callback): # 同步future
        print (callback, '+++++++++++++++') # d -> 对应log 的d
        callback(42)

    @return_future
    def async_future(self, callback): #异步future
        print (callback, '+++++++++++++++') # d -> 对应 log 的 d
        self.io_loop.add_callback(callback, 42)
    
    def test_sync_future(self): #测试同步 
        future = self.sync_future()
        self.assertEqual(future.result(), 42)

    def test_async_future(self): #测试异步
        future = self.async_future()
        self.assertFalse(future.done())
        self.io_loop.add_future(future, self.stop) #add_future
        future2 = self.wait()
        self.assertIs(future, future2)
        self.assertEqual(future.result(), 42)
```

联合 concurrent.py -> return_future


```python
def return_future(f):
    replacer = ArgReplacer(f, 'callback') # 1 

    @fuctools.wraps(f)
    def wrapper(*args, **kwrags):
        print (args, kwrags, "argsssssssss") #a 对应下面的 log -> a
        future = TracebackFuture() # 2
        callback, args, kwargs = replacer.replace(
            lambda value=_NO_RESULT: future.set_result(value),
            args, kwargs) # 1  替代 f 函数的 `callback`,  
        print (callback, args, kwargs, 'callback') #b 对应下面的 log -> b

        def handle_error(typ, value, tb):
            future.set_exc_info((typ, value, tb))
            return True

        exc_info = None
        with ExceptionStackContext(handle_error): # 4
            try:
                result = f(*args, **kwargs) 
                print (result, 'result-------------------') #c 对应下面的 log -> c
                if result is not None:
                        raise ReturnValueIgoredError(
                            "@return_future should not be used with functions"
                            "that return values")
            except:
                exc_info = sys.exc_info()
                raise
        if exc_info is not None:
            raise_exc_info(exc_info)

        if callback is not None:
            def run_callback(future):
                result = future.result()
                print (future, "+__+_+_+_+_") #e -> 对应下面 ＬＯＧ -> e
                if result is _NO_RESULT:
                    callback()
                else:
                    callback(future.result())
            future.add_done_callback(wrap(run_callback)) # 6
        return future
    return wrapper
```

###分几种情况 1. 同步无回调 2.同步有回调 3.异步无回调 4.异步有回调
 - **同步无回调LOG**  `f(args, kwargs): future = sync_process()`
   - *a* --  (<test_return_future.ReturnFutureTest testMethod=test_no_callback>,) {} argsssssssss
   - *b* -- None (<test_return_future.ReturnFutureTest testMethod=test_no_callback>,) {'callback': <function <lambda> at 0xb6ba8f0c>} callback
   - *c* -- None result------------------ 
   - *d* -- (function <lambda> at 0xb6ba8f0c>) +++++++++++++

> 从 a-b 可以理解 replacer.replace 的作用: 提取callback 的值， 并将callback 放入kwargs; 由c 可以知道 f() 函数是不会return 的; f()的结果只能由 future.result() 得到, 只要知道reuturn_future 是返回Future 本身是不会return 的， 如果return 就会报错; 由d 可知匿名函数`fuction <lambda> at 0xb6ba8f0c`赋值给了callback,而这个匿名函数的作用就是`set_result`。

 - **同步有回调LOG** `f(args, kwargs, callback): future = sync_process() callback(future)`
   - *a* -- (<test_return_future.ReturnFutureTest testMethod=test_callback_kw>,) {'callback':<bound method ReturnFutureTest.stop of <test_return_future.ReturnFutureTest testMethod=test_callback_kw>>} argsssssssss
   - *b* -- <bound method ReturnFutureTest.stop of <test_return_future.ReturnFutureTest testMethod=test_callback_kw>> (<test_return_future.ReturnFutureTest testMethod=test_callback_kw>,) {'callback': <function <lambda> at 0xb6bd8f44>} callback  
   - *c* --  None result------------------
   - *d* -- 额额 function <lambda> at 0xb6bd8f44 +++++++++++++ 
   - *e* -- 额额 Future at 0xb6b9cc8cL state=finished returned int +__+_+_+_+_ 

> 与无回调相比; 明显多出 e log; run_callback 就是将 future.result()作为callback的参数运行; 
 
 - **异步无回调LOG** `f(args, kwargs): future = async_process()`
  - *a* -- (<test_return_future.ReturnFutureTest testMethod=test_async_future>,) {} argsssssssss 
  - *b* -- None (<test_return_future.ReturnFutureTest testMethod=test_async_future>,) {'callback': <function <lambda> at 0x8518f44>} callback 
  - *c* -- None result------------------ 
  - *d* -- (function <lambda> at 0x8518f44) ++++++++++++++ 

> 与同步无回调相比; 看`test_async_future(self)`, 将 f() 获得的 future -> self.io_loop.add_future(future, self.stop) -> self.stop(future) -> 最后通过 self.wait()获得的结果 就是 例子中42 . 现实中一般将 return_future 与 gen.engine 联合使用  , 通过在 gen.engine -> yield f() 获得结果


