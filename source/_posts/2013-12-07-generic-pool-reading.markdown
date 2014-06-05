---
layout: post
title: "generic-pool Reading"
date: 2013-12-07 02:00
comments: true
categories: codeReading
---

##[node-pool](https://github.com/coopernurse/node-pool)

###why pool
 - 与数据建立连接关闭连接， 每次建立一个连接对象会有消耗!(在并发很高的情况消耗会更明显)

###what node-pool can do ?
 - 动态调整连接数的连接池, 数据库频繁读写时, 就建多个连接, 当然连接数有最大值Max, 如果读写实际需要的连接数 > Max 那么加入队列里;如果数据库读写空闲，就释放多余的连接;

###how node-pool do that ? -> Code Reading
 - 栗子
<script src="https://gist.github.com/zs1621/7836578.js"></script>


运行上面的例子

 1. 新建一个pool对象, 初始化一些变量
    - idleTimeoutMillis: 一个连接空闲时间最大值, 如果设置为30000, 那么一个连接空闲30000ms 后会自动关闭, 默认 30000ms
    - reapIntervalMillis:  每`reapIntervalMillis`检查空闲并移除, 默认1000ms
    - max: 连接池存在的最大连接数, 例子里设置为10
    - min: 连接池存在的最小连接数, 例子里默认为0(备注: 这里有个疑问，当我将min设为1, 从mongodb log 中可以看到 在空闲时间 连接是每隔`idleTimeoutMillis`关闭并新建连接, 问题是为什么不保持一个连接而要不停的关闭新建, 如果这样不如)
    - log: 可以自定义node-pool, 例子直接设为true 默认用node-pool提供的log 
    - create: 应该创建一个item(db) 被 `acquired`, 然后调用其创建的 item 作为参数
    - destory: 在items毁之前,关闭所有资源使用的item(db) 
 2. ensureMinimum, 确保有最小连接数
   - 例子中如果给min赋一个大于0的值, 2, 那么就 createResource()两次 
   - 因为例子默认为0 跳过
 3. 至此已经建立pool, 下一步等待`acquired`, 资源准备就续, 就等消费了 
 4. `curl http://127.0.0.1:8080`
 5. `acquire(callback, priority)`  
    - waitingClients.enqueue(callback, priority): 将回调根据priority推入队列
    - dispense(): 由单词意思可知分配资源， 施行; 试着拿一个客户端工作， 清除空闲的item. 如果有等待的client, shift(), call its callback. 如果没有等待的client, 创建一个; 如果创建一个client将超过max, 把客户端加入等待list!代码见 [dispense](#dispense)
   - dispense() 结束,此时 waitingClients.size() = 0, availableObjects.length = 0
 6. release(): 如果client不再需要，将之返回pool, 代码见[release](#release)  
 7. removeIdle(): check and removes the available clients that have timed out. 见[removeIdle](#removeIdle)

 8. me.destroy(): client to be destroyed.见[destroy](#destroy)

<a name="dispense" id="despense">**dispense**</a>
```javascript
    function dispense() {
        var obj = null, 
        objWithTimeout = null,
        err = null,
        clientCb = null,
        waitingCount = waitingClients.size();
        log("...");
        if (waitingCount > 0) { //此时waitingCount = 1
            while(availableObjects.length > 0) {
                log("...");    
                objWithTimeout = avaiableObjects[0];
                if (!factory.validate(objWithTimeout.obj)) {
                    me.destroy(objWithTimeout.obj);     
                    continue;
                } // 这一步验证对象
                avaiableObjects.shift(); // LIFO
                clientCb = waitingClients.dequeue(); // dequeue
                return clientCb(err, objWithTimeout.obj); // callback
            }     
            if (count < factory.max) {
                    createResource();
            }
        }
    }
```

<a name="release" id="release">**release**</a>
```javascript
me.release = function(obj) {
    // 确保对象已经被释放
    if (availableObjects.some(function(objWithTimeout) {
        return (objWithTimeout.obj === obj); 
        })) {
        log("release called twice for the same resource")
        return;
    }     
    var objWithTimeout = { obj: obj, timeout: (new Date().getTime() + idleTimeoutMillis) };
    availableObjects.push(objWithTimeout); // 此时 availableObjects.size()=1
    log("timeout:..");
    dispense();  
    scheduleRemoveIdle(); // 计划移走idle items(空闲项目) 延迟1000ms 执行 removeIdle
}
```


<a name="removeIdle" id="removeIdle">**removeIdle**</a>
```javascript
 function removeIdle(){
    var toRemove = [],
        now = new Date().getTime(),
        i,
        al, tr,
        timeout;
    removeIdleScheduled = false;
    // go through the available(idle) items, check if they have timed out;
    for (i=0, al=availableObjects.length; i< al &&
        (refreshIdle) || (count - factory.min > toRemove.length));i+=1
        ){
        timeout = availableObjects[i].timeout; 
        if (now >= timeout) {
           //client timed out, so destroy it 
           toRemove.push(availableObjects[i].obj);
            }
        }
        for (i=0, tr=toRemove.length; i<tr; i+=1) {
            me.destroy(toRemove[i]);     
        }
        al = availableObjects.length;
        if (al > 0) {
           log(".."); 
           scheduleRemoveIdle();
        } else {
            log("removeIdle() all objects removed", 'verbose');     
        }
     }
```

<a name="destroy" id="destroy">**destroy**</a>
```javascript
 me.destroy = function(obj) {
    count -= 1; 
    availableOjbects = availableObjects.filter(function(objWithTimeout) {
       return (objWithTimeout.obj != obj) 
        });//过滤掉return true 的对象
    factory.destroy(obj);
    ensureMinimum();
     }
```


