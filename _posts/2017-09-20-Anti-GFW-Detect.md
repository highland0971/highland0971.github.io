---
layout: default
title:  "基于无预置IP list的GFW IP解锁方法"
date:   2017-09-20 16:16:01 +0800
categories: GFW
---

## 基于无预置IP list的GFW IP解锁方法

在以往的基于路由、iptables解锁GFW IP封锁的方法中，均需要一份国内或国外的IP地址列表，然后进行策略路由。基于这种方案的解锁，需要人工去维护一张IP地址列表，其中基于[chnroutes](https://github.com/fivesheep/chnroutes)方案生成的国内IP地址列表就有8000多条记录,在加载的时候会比较耗时，同时给路由器带来一定负担。本文给出一种基于无IP列表的自动IP访问监测及路由更变策略。

1. 修改iptables规则，实现对封锁IP或者干扰IP的自动检测标记
- 封锁 IP监测，对于被封锁的IP，客户端往往会在短时间内多次发起TCP SYN请求，通过捕获高频SYN请求，识别潜在的被封锁IP地址
```bash
iptables -I FORWARD -i br0 -o eth0 -p tcp -m tcp --tcp-flags SYN,ACK SYN \
        -m recent --rcheck --seconds 15 --hitcount 3 --name syn_watch_list --rdest \
        -j LOG --log-prefix "GFW_DECT_SYN "
iptables -I FORWARD -i br0 -o eth0 -p tcp -m tcp --tcp-flags SYN,ACK SYN \
        -m recent --set --name syn_watch_list --rdest
iptables -I FORWARD -i br0 -o eth0 -p tcp -m tcp --tcp-flags SYN,ACK SYN \
        -j LOG --log-prefix "GFW_DEBUG "
```
- 受干扰IP监测（敏感词过滤），对于被干扰的IP，通过捕获来自WAN口的RST请求，识别潜在被封锁的IP地址
```bash
iptables -I FORWARD -i eth0 -p tcp -m tcp --tcp-flags RST RST -j LOG --log-prefix "GFW_DECT_RST "
```

2. 将监测到的被干扰IP加入策略路由或iptables端口重定向(基于**ss-redir**实现)
```bash
export GFW_LOCAL_GATEWAY=[Replace with your free Internat access gateway ip]
export GFW_REMOTE_VPN=[Replace with your remote ss-server ip]

watch -n 1 dmesg -c|grep GFW_DECT|grep -v $GFW_REMOTE_VPN|\
        awk '/GFW_DECT_SYN/{print $5};/GFW_DECT_RST/{print $4}'|\
        awk -F= '{if($1=="SRC"){cmd="route -n|grep "$2" >/dev/null 2>&1 || \
                route add "$2" gateway "ENVIRON["GFW_LOCAL_GATEWAY"]" >/dev/null 2>&1"} \
                else{cmd="route -n|grep "$2" || ping -c 1 -W 1 "$2"  >/dev/null 2>&1 || \
                route add "$2" gateway "ENVIRON["GFW_LOCAL_GATEWAY"]" >/dev/null 2>&1"}; \
                print(cmd);system(cmd)}'
```

3. 在以上的配置中，是将识别出来的受干扰IP路由给了另外一个装着**shadowsocks**的路由器，由其使用**ss-redir**进行透明转发
配置如下：
```bash
export GFW_LOCAL_GATEWAY=[Replace with your free Internat access gateway ip]
export GFW_REMOTE_VPN=[Replace with your remote ss-server ip]
sudo ss-nat -s $GFW_REMOTE_VPN -l [Your ss-redir local listening port] -b $GFW_REMOTE_VPN -u
sudo nohup ss-redir -s $GFW_REMOTE_VPN -p [Your ss-server listening port] \
                    -k [Your ss-server password] -m [Your ss-server crypto method] \
                    -l [Your ss-redir local listening port] -b 0.0.0.0 -u
```
