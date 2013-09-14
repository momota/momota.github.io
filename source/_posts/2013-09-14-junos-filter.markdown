---
layout: post
title: "junos firewall filter term 定義を挿入する方法"
date: 2013-09-14 12:36
comments: true
categories: junos juniper filter network security
---
![junos](http://upload.wikimedia.org/wikipedia/en/8/86/Junos_sw_logo.jpg)


junos の firewall filter の設定で定義の順序について。(EXシリーズを想定)

以下、すべてconfiguration mode での操作。

下記のようなフィルタがすでに設定されている場合に、`TERM10` と `TERM20` の間に `TERM15` を挿入したいことがある。フィルタは上から順に評価されるし。

```
# show | display set 
...
set firewall family inet filter FILTER01 term TERM10 from source-address 10.0.0.1/32
set firewall family inet filter FILTER01 term TERM10 from destination-address 192.168.1.0/24
set firewall family inet filter FILTER01 term TERM10 from tcp-established
set firewall family inet filter FILTER01 term TERM10 then accept
set firewall family inet filter FILTER01 term TERM20 then count COUNTER20
set firewall family inet filter FILTER01 term TERM20 then discard
...
```

<!-- more -->

単純に以下のターム定義 `TERM15` を追加で設定すると

```
# set firewall family inet filter FILTER01 term TERM15 from source-address 10.0.0.2/32
# set firewall family inet filter FILTER01 term TERM15 from destination-address 192.168.1.0/24
# set firewall family inet filter FILTER01 term TERM15 from protocol tcp
# set firewall family inet filter FILTER01 term TERM15 from destination-port 20-21
# set firewall family inet filter FILTER01 term TERM15 then accept
```

以下のように、意図した箇所に設定が反映されない。
つまり、10.0.0.2/32 から 192.168.1.0/24 への FTP 通信は許可されない。

```diff
# show | display set 
...
  set firewall family inet filter FILTER01 term TERM10 from source-address 10.0.0.1/32
  set firewall family inet filter FILTER01 term TERM10 from destination-address 192.168.1.0/24
  set firewall family inet filter FILTER01 term TERM10 from tcp-established
  set firewall family inet filter FILTER01 term TERM10 then accept
  set firewall family inet filter FILTER01 term TERM20 then count COUNTER20
  set firewall family inet filter FILTER01 term TERM20 then discard
+ set firewall family inet filter FILTER01 term TERM15 from source-address 10.0.0.2/32
+ set firewall family inet filter FILTER01 term TERM15 from destination-address 192.168.1.0/24
+ set firewall family inet filter FILTER01 term TERM15 from protocol tcp
+ set firewall family inet filter FILTER01 term TERM15 from destination-port 20-21
+ set firewall family inet filter FILTER01 term TERM15 then accept
...
```

`TERM10` と `TERM20` の間に `TERM15` を挿入する場合は、以下のように既存定義をすべて `delete` してから新しい定義追加済みの定義を設定し直す必要があるが、ターム定義が1000くらいあるケースを想定するとそれは避けたい。

```
# delete firewall family inet filter FILTER01 term TERM10
# delete firewall family inet filter FILTER01 term TERM20
# 
# set firewall family inet filter FILTER01 term TERM10 from source-address 10.0.0.1/32
# set firewall family inet filter FILTER01 term TERM10 from destination-address 192.168.1.0/24
# set firewall family inet filter FILTER01 term TERM10 from tcp-established
# set firewall family inet filter FILTER01 term TERM10 then accept
# set firewall family inet filter FILTER01 term TERM15 from source-address 10.0.0.2/32
# set firewall family inet filter FILTER01 term TERM15 from destination-address 192.168.1.0/24
# set firewall family inet filter FILTER01 term TERM15 from protocol tcp
# set firewall family inet filter FILTER01 term TERM15 from destination-port 20-21
# set firewall family inet filter FILTER01 term TERM15 then accept
# set firewall family inet filter FILTER01 term TERM20 then count COUNTER20
# set firewall family inet filter FILTER01 term TERM20 then discard
```

このようなときには `insert` を使う。以下のようにターム設定後に `insert` してあげればOK。


```
# set firewall family inet filter FILTER01 term TERM15 from source-address 10.0.0.2/32
# set firewall family inet filter FILTER01 term TERM15 from destination-address 192.168.1.0/24
# set firewall family inet filter FILTER01 term TERM15 from protocol tcp
# set firewall family inet filter FILTER01 term TERM15 from destination-port 20-21
# set firewall family inet filter FILTER01 term TERM15 then accept
#
# insert firewall family inet filter FILTER01 term TERM15 after term TERM_10
```

そうすると、以下のように意図したとおり定義できる。

```diff
# show | display set 
...
  set firewall family inet filter FILTER01 term TERM10 from source-address 10.0.0.1/32
  set firewall family inet filter FILTER01 term TERM10 from destination-address 192.168.1.0/24
  set firewall family inet filter FILTER01 term TERM10 from tcp-established
  set firewall family inet filter FILTER01 term TERM10 then accept
+ set firewall family inet filter FILTER01 term TERM15 from source-address 10.0.0.2/32
+ set firewall family inet filter FILTER01 term TERM15 from destination-address 192.168.1.0/24
+ set firewall family inet filter FILTER01 term TERM15 from protocol tcp
+ set firewall family inet filter FILTER01 term TERM15 from destination-port 20-21
+ set firewall family inet filter FILTER01 term TERM15 then accept
  set firewall family inet filter FILTER01 term TERM20 then count COUNTER20
  set firewall family inet filter FILTER01 term TERM20 then discard
```

