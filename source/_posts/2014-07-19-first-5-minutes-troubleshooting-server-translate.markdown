---
layout: post
title: "First 5 minutes troubleshooting server translate "
date: 2014-07-19 17:09:13 +0800
comments: true
categories: translation
---

 > [原文](http://devo.ps/blog/troubleshooting-5minutes-on-a-yet-unknown-box/)


##**服務器遇到問題,從何下手?**


不要急着衝向你的服務器， 在這之前， 你對它和具體的問題了解多少？ 


必須知道的
 
 - 問題的確切症狀? 丟包? 錯誤?
 - 問題什麼時候被發現?
 - 可以重現麼?
 - 有什麼規律(例如每小時發生一次)?
 - 在服務器的最後一次改變(代碼， 服務配置, 內存)?
 - 問題會影響客戶的哪個部分(登錄， 註銷， )?
 - **有監控平臺麼?** Munin, Zabbix, Nagios, NewRelic.. 
 - **日志(集中化)?** Loggly, Airbrake, Graylog



上面的最後兩項是最方便直觀的信息來源， 但也別期望太多;爲了搞定問題，繼續往下看


**誰登陸了?**


```
$ w
$ last
```

不是很重要， 但是你在找問題時不希望還有其他人在擺弄的服務器。廚房裏一個廚師足矣


**之前做了哪些操作?**


```
$ history
```

看歷史記錄總是好的;配合之前誰在操作的命令.


快速回憶一下， 你也許小要升級環境變量 `HISTTIMEFORMAT` 去跟蹤那些命令運行的時刻.沒什麼比調查一個過期命令列表更讓人沮喪的了。


**正在運行哪些程序i?**


```
$ pstree -a
$ ps aux
```


如果覺得`ps aux`太羅嗦， `pstree -a`會給你一個漂亮的展示關於什麼正在運行以及誰調用了它


**監聽服務**


```
$ netstat -ntlp  # t: tcp
$ netstat -nulp  # u: udp
$ netstat -nxlp  # x: unix socket
$ netstat -ant|grep EST|wc -l #这条命令是很有用的能看到多少连接
$ netstat -ant|wc -l  #看到有多少tcp,udp链接
```

我更傾向分開運行這些命令， 主要是因爲不喜歡同時看所有的服務。雖然`netstat -nalp`以可以做到。甚至我會省略`數字化`選項(IPs是易讀點的)


確定正在運行的服務以及他們是否應該運行。尋找多個監聽端口。通過`ps aux`的輸出匹配進程ID；這個是很有用的特別當你同時結束2~3個Java 或者 Erlang進程


**CPU 和 RAM**


```
$ free -m
$ uptime
$ top
$ htop
```

這些命令回答以下問題:
 -  還有空閒的內存? 還是正在交換區 ?
 -  還有CPU剩餘麼? 服務器有多少CPU核心可用? 這些核心中是否有超載的?
 -  哪個項目負載最多? 平均負載多少?
 -  詳細了解命令`man`,你懂的
 -  这里还有一点如果cpu不高，但是服务里面响应时间很大，那么就是程序里的代码卡住，怎么找出哪里卡住，可以通过`strace -p yourpid`先查看你的程序pid然后用此命令


**Hardware**


```
$ lspci
$ dmidecode
$ ethtool
```


TODO: explain commands


**IO 性能**


```
$ iostat -kx 2
$ vmstat 2 10
$ mpstat 2 10
$ dstat --top-io --top-bio
```

這幾個是非常有用的命令去分析後臺性能;

 - check 硬盤使用: 
 - 交換區目前有沒有使用
 - 什麼程序再使用CPU: 系統? 用戶? stolen ?
 - `dstat` 一直是我的最愛. IO消耗? mysql 阻塞資源? php進程在使用的你io...


**掛載點和文件系統**


```
$ mount
$ cat /etc/fstab
$ vgs
$ pvs
$ lvs
$ df -h
$ lsof +D / /* 
```


 - 多少文件系統被掛載
 - 過期的文件系統
 - 硬盤空間是否有剩餘
 - 大文件還沒有清除
 - 如果硬盤有問題還有空間去加個分區?


 **內核,中斷, 網絡使用情況**


```
$ sysctl -a | grep ...
$ cat /proc/interrupts
$ cat /proc/net/ip_conntrack
$ netstat
$ ss -s
```
TBC


