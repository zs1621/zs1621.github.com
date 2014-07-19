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
```

我更傾向分開運行這些命令， 主要是因爲不喜歡同時看所有的服務。雖然`netstat -nalp`以可以做到。甚至我會省略`數字化`選項(IPs是易讀點的)


確定正在運行的服務以及他們是否應該運行。尋找多個監聽端口。通過`ps aux`的輸出匹配進程ID；這個是很有用的特別當你同時結束2~3個Java 或者 Erlang進程


TBC


