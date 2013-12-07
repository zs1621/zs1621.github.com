---
layout: post
title: "generic-pool Reding"
date: 2013-12-07 02:00
comments: true
categories: 
---

##[node-pool](https://github.com/coopernurse/node-pool)

###why pool
 - 与数据建立连接关闭连接， 每次建立一个连接对象会有消耗!(在并发很高的情况消耗会更明显)

###what node-pool can do ?
 - 动态调整连接数的连接池, 数据库频繁读写时, 就建多个连接, 当然连接数有最大值Max, 如果读写实际需要的连接数 > Max 那么加入队列里;如果数据库读写空闲，就释放多余的连接;

###how node-pool do that ? -> Code Reading
 - 栗子
<script src="https://gist.github.com/zs1621/7836578.js"></script>
TBC

