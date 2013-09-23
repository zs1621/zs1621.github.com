---
layout: post
title: "Tornado Code Reading - IOLoop"
date: 2013-09-19 12:36
comments: true
categories: 
---

##[IOLoop](https://github.com/facebook/tornado/blob/master/tornado/ioloop.py)

对着IOLoop 的源码瞅了一天，楞是没有明白， 为什么 父类实例可以调用子类方法?

最后看到[configurable](http://hswg.info/blog/2013/03/24/configurable-of-tornado-note/)

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
当子类对象被构造时， 子类__new__被调用， 因此参数里的cls 指的是Configurable的子类(可配置接口类， 如IOLoop)。先得到base,  [IOLoop代码](#IOLoop) 可知, configurable_base返回的是自身类。由于 base 和 cls 是一样的， 所以调用 configured_class() 得到接口的子类实现(见[configured_class](#configured_class)) 其实就是调用 base的 configurable_default, 就是返回一个子类实现(epoll/kqueue/select之一),顺便把__impl_kwargs合并到args 里 。然后调用Configurable类的父类(Object)的 __new__方法， 生成一个impl的对象， 紧接着把args当参数调用该队想的initialize(继承PollIOLoop) , 返回该对象。 所以， 当构造IOLoop对象时， 实际得到的是EPollIOLoop或其它相关子类。可以看出， Configurable 类主要提供构造方法， 相当于对象工厂根据配置来生产对象， 同时开放configure接口以供配置。而子类按照约定调整配置即可得到不同对象， 代码得到了复用 或其它相关子类。可以看出， Configurable 类主要提供构造方法， 相当于对象工厂根据配置来生产对象， 同时开放configure接口以供配置。而子类按照约定调整配置即可得到不同对象， 代码得到了复用 

<a name="configured_class" id="configured_class">configured_class</a>
```
base = cls.configurable_base()
if cls.__impl_class is None:
	base._impl_class = cls.configurable_default()
return base.__impl_class
```

下面来看 IOLoop 的对象 instance
