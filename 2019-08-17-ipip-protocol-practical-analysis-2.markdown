---
layout:     post
title:      "IPIP协议实例分析《二》"
subtitle:   "配置ipip实例，分析ipip协议封包，解包，路由的过程"
date:       2019-08-17
author:     "ChenWenKe"
tags:
	- TCP/IP
	- 网络编程
mathjax: true
---

​        承接上一篇博客，假定我们已经建立好了机器A和机器B之间的IPIP隧道，也已经为机器A端的隧道，机器B端的隧道分配好了IP。

​		现在，我们如果在机器A上访问机器B的eth0会怎样？根据路由表，当然是直接通过eth0出去，然后得到应答，应答数据包从eth0进来了。但是如果我们指定从tun_a出去，会怎么样？

#### 机器A上通过IPIP隧道访问机器B的eth0

```bash
[root@VM_16_22_centos]# ping -I tun_a 172.16.1.62
PING 172.16.1.62 (172.16.1.62) from 192.168.1.1 tun_a: 56(84) bytes of data.


```

咦~~， 没有ICMP的应答报文。

抓包查看一下：

- 抓取机器A tun_a上的数据包：

  ```
  [root@VM_16_22_centos]# tcpdump -i tun_a -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on tun_a, link-type RAW (Raw IP), capture size 262144 bytes
  15:28:54.807858 IP 192.168.1.1 > 172.16.1.62: ICMP echo request, id 31060, seq 229, length 64
  15:28:54.809458 IP 172.16.1.62 > 192.168.1.1: ICMP echo reply, id 31060, seq 229, length 64
  15:28:55.807866 IP 192.168.1.1 > 172.16.1.62: ICMP echo request, id 31060, seq 230, length 64
  15:28:55.809483 IP 172.16.1.62 > 192.168.1.1: ICMP echo reply, id 31060, seq 230, length 64
  ```

  神奇的是 tun_a 上即抓到了ICMP的echo 请求包，又抓到了 ICMP的echo应答包。tun_a上既有请求包，又有应答包，理所当然，eth0上也绝对有请求包和应答包。

- 抓取机器A eth0上的数据包：

  ```bash
  [root@VM_16_22_centos ]# tcpdump -i eth0 not tcp -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
  15:33:05.808890 IP 172.16.16.22 > 172.16.1.62: IP 192.168.1.1 > 172.16.1.62: ICMP echo request, id 31060, seq 480, length 64 (ipip-proto-4)
  15:33:05.810527 IP 172.16.1.62 > 172.16.16.22: IP 172.16.1.62 > 192.168.1.1: ICMP echo reply, id 31060, seq 480, length 64 (ipip-proto-4)
  ```



到底发生了什么呢？ 为什么机器A明明收到了ICMP应答数据包，但是ping进程却没收到呢。



再度回顾一下机器A处理ICMP应答数据包的过程：

- ICMP应答数据包到达机器A的eth0: `IP 172.16.1.62 > 172.16.16.22: IP 172.16.1.62 > 192.168.1.1`, 把数据包提交给协议栈。
- 协议栈解析ICMP应答数据包，发现是IPIP数据包，把数据包发送给tun_a。
- tun_a中接到了`IP 172.16.1.62 > 192.168.1.1`的ICMP应答数据包，把数据包提交给协议栈。
- 按理说协议栈此时应该查询相关的套接字对，把ICMP应答数据包拷贝到该套接字的缓冲区才对。但是协议栈并没有这么做，这是为什么呢？

查看一下tun_a的路由表:

```bash
[root@VM_16_22_centos ]# ip route show dev tun_a
192.168.1.0/24 proto kernel scope link src 192.168.1.1
```

发现tun_a中接到的`IP 172.16.1.62 > 192.168.1.1`的ICMP应答数据包的源地址和`192.168.1.0/24`并不匹配，是的呢，我们是`ping -I tun_a`指定tun_a把ICMP请求报文发出去的，默认情况下按照系统路由表数据包并不会经过tun_a， 而是会直接从eth0出去。而Linux中，为了安全起见，是对网络接口的路由回包有所限制的。



#### rp_filter:  reverse path filtering

​        rp_filter是对回包进行过滤的一种机制，当rp_filter开启的时候，会对每个网络接口接收到的回包进行校验，看一下这个包是否是从这个网络接口出去的(即: 检查回包的源IP和该网络接口的路由表是否匹配)，如果是，则接收这个数据包；如果不是，则直接丢弃这个数据包。当rp_filter关闭时，不对回包的源IP进行检查。

​      红帽的实现(Ret Hat 或CentOS )，提供了第三种选项，把rp_filter设置为2，对回包进行严查时，如果这个包是从这台机器的任意网络接口发出去的，都接收这个回包数据，否则丢弃这个回包数据。



在机器A上关闭rp_filter或者设置rp_filter为2：

```bash
 # 关闭rp_filter检查
 echo 0 > /proc/sys/net/ipv4/conf/tun_a/rp_filter
 echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter
 # 或者设置rp_filter为2
 echo 2 > /proc/sys/net/ipv4/conf/tun_a/rp_filter
 echo 2 > /proc/sys/net/ipv4/conf/all/rp_filter
```



这时, 在机器A上通过IPIP隧道ping 机器B的eth0, socket就能收到ICMP的应答包了。

```
[root@VM_16_22_centos]# ping -I tun_a 172.16.1.62
PING 172.16.1.62 (172.16.1.62) from 192.168.1.1 tun_a: 56(84) bytes of data.
64 bytes from 172.16.1.62: icmp_seq=1 ttl=64 time=1.61 ms
64 bytes from 172.16.1.62: icmp_seq=2 ttl=64 time=1.63 ms
```



#### 参考资料

- [Linux kernel rp_filter settings (Reverse path filtering )](https://www.slashroot.in/linux-kernel-rpfilter-settings-reverse-path-filtering)