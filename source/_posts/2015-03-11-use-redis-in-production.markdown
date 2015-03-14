---
layout: post
title: "use redis in production - redis config set"
date: 2015-03-11 13:58:23 +0800
comments: true
categories: NoSQL 
---

##redis config set

 - since redis 2.0.0, u can use this change setting in redis running
 - config set可以修改哪些配置: `config get *` 



###注意事项
 - config set 只能在redis运行时改变配置，但不能改变配置文件里的配置，所以当你在redis运行时，使用了config set，必须在对应的配置文件里做修改，否则下次启动时配置又会改变
 


###实战
 - 最近的项目里需要对redis数据库做迁移,从系统盘->数据盘,场景是只使用了RDB持久化
 - 使用config set dir 需注意

```  
rdb将内存中的数据库以RDB格式保存到磁盘中，如果RDB文件已存在，那么新的RDB文件写好将会替换为已有的RDB文件。
在保存RDB文件期间，主进程会被调用rdbsave函数 
bgsave过程fork出一个子进程，子进程负责调用rdbSave,并在保存完成之后向主进程发送信号，通知保存已完成
save过程直接调用rdbSave,阻塞Redis主进程，知道保存完为止。在主进程阻塞期间，服务器不能处理客户端的任何请求。
这里有两个坑
1. fork是有延迟的 [](http://blog.nosqlfan.com/html/3903.html)
2. RDB在写入过程中，会连内存一起Fork出一个新进程，遍历新进程内存中的数据写RDB. 注意fork的时候一定要注意当前机子的内存，保险的话是2倍现有redis所占内存.
```

  - dump.rdb 从 /var/lib/redis/ ， 转存到/mnt/lib/redis/
  - redis-cli
  - config set dir /mnt/lib/redis #这里要注意redis进程是属于哪个用户的, 要把/mnt/lib/redis设为此用户可写可读的
  - bgsave  #当看到/mnt/lib/redis 里有dump.rdb(默认)文件时就表明操作成功
  - 此时再修改 redis.conf 将 dir -> /mnt/lib/redis
  - 如果下次重启的话， redis将会从 /mnt/lib/redis/dump.rdb 读入数据到内存中  #注意如果要重启，最好先把业务逻辑相关的redis处理关掉，防止数据丢失，并且通过shutdown 关闭，不要kill -9。


### config set 还可以修改很多参数这里不一一解释等以后用到再做解释TBC    
  

