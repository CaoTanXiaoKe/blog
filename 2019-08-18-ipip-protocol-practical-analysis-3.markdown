---
layout:     post
title:      "IPIP协议实例分析《三》"
subtitle:   "配置ipip实例，分析ipip协议封包，解包，路由的过程"
date:       2019-08-18
author:     "ChenWenKe"
tags:
	- TCP/IP
	- 网络编程
mathjax: true
---



​         承接上两篇博客，假定我们已经建立好了机器A和机器B之间的IPIP隧道，也已经为机器A端的隧道，机器B端的隧道分配好了IP。



#### 机器A通过隧道访问外网(qq.com)

​		现在，我们如果在机器A上通过隧道访问qq.com(180.163.26.39)会怎么样(假设机器B可以访问Internet)？ 根据上两篇的分析，我们目前很容易知道，从隧道tun_a出去，会选择tun_a的IP作为源IP，因此内层IP为： `192.168.1.1 > 180.163.26.39`, 经过IPIP隧道需要封装一层IP header: `172.16.16.22 > 172.16.1.62`。

​		根据外层IP的路由规则，目标IP为`172.16.1.62`, 因此数据包会发送到机器B的eth0。



#### 机器B上的处理过程。

- 机器B上eth0收到数据包，把数据包转交给协议栈，协议栈发现数据包是IPIP数据包，对数据包进行解包, 取出原IP数据包，转给tun_b， tun_b 把数据提交给协议栈，根据目的IP匹配路由，发现目的IP`qq.com(180.163.26.39)` 不是本机网络接口的IP，默认情况下丢弃数据包。

  分别对eth0和tun_b进行抓包查看验证：

  ```bash
  [root@VM_1_62_centos ~]# tcpdump -i eth0 not tcp -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
  00:28:16.170139 IP 172.16.16.22 > 172.16.1.62: IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 26, length 64 (ipip-proto-4)
  ```

  ```bash
  [root@VM_1_62_centos ~]# tcpdump -i tun_b not tcp -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on tun_b, link-type RAW (Raw IP), capture size 262144 bytes
  00:29:50.172210 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 120, length 64
  00:29:51.172243 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 121, length 64
  00:29:52.172223 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 122, length 64
  00:29:53.172204 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 123, length 64
  00:29:54.172188 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 124, length 64
  00:29:55.172251 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 125, length 64
  ```

- 为了解决机器drop目的地址不是本机IP的问题, 开启ip_forword设置。

  ```bash
  # 开启IPv4转发
  echo 1 > /proc/sys/net/ipv4/ip_forward
  ```

  

- 抓包查看

  ```bash
  [root@VM_1_62_centos ~]# tcpdump -i eth0 src host 192.168.1.1 -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
  00:35:55.174225 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 485, length 64
  00:35:56.174342 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 486, length 64
  00:35:57.174208 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 487, length 64
  00:35:58.174205 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 30617, seq 488, length 64
  ```

  根据抓包结果，可以看到开启ip_forword选项后，目的IP为`qq.com(180.163.26.39)`的数据包，的确是从eth0转发出去了。但是没有收到回包。

- 上一步中发出去的ICMP echo request数据包为`IP 192.168.1.1 > 180.163.26.39`, 源IP为机器A的私有网络IP，因此回包的时候无法正确的回到机器A或者机器B。此时需要把`IP 192.168.1.1 > 180.163.26.39`的源IP转换成机器B上的公网IP(我们的例子，即`eth0172.16.1.62`) 因此需要在数据包出去的时候，需要执行snat。

  ```bash
  iptables -t nat -A POSTROUTING -j SNAT -o eth0 --to-source 172.16.1.62
  ```

- 分别从eth0和tun_b上抓包

  ```bash
  [root@VM_1_62_centos ~]# tcpdump -i eth0 src host 180.163.26.39 -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
  00:49:13.581165 IP 180.163.26.39 > 172.16.1.62: ICMP echo reply, id 546, seq 32, length 64
  00:49:14.581145 IP 180.163.26.39 > 172.16.1.62: ICMP echo reply, id 546, seq 33, length 64
  ```

  ```bash
  [root@VM_1_62_centos ~]# tcpdump -i tun_b not tcp -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on tun_b, link-type RAW (Raw IP), capture size 262144 bytes
  00:49:24.552199 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 546, seq 43, length 64
  00:49:24.581165 IP 180.163.26.39 > 192.168.1.1: ICMP echo reply, id 546, seq 43, length 64
  00:49:25.552197 IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 546, seq 44, length 64
  00:49:25.581175 IP 180.163.26.39 > 192.168.1.1: ICMP echo reply, id 546, seq 44, length 64
  ```

  ```bash
  [root@VM_1_62_centos ~]# tcpdump -i eth0 not tcp -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
  00:52:19.552232 IP 172.16.16.22 > 172.16.1.62: IP 192.168.1.1 > 180.163.26.39: ICMP echo request, id 546, seq 218, length 64 (ipip-proto-4)
  length 64
  00:52:19.581264 IP 172.16.1.62 > 172.16.16.22: IP 180.163.26.39 > 192.168.1.1: ICMP echo reply, id 546, seq 218, length 64 (ipip-proto-4)
  ```



####在机器A上查看ping进程回显结果

如果机器A上ping进程没有会显ICMP echo reply的数据报文，参考上一篇博客，因为到达tun_a的原IP数据报为: 

`IP 180.163.26.39 > 192.168.1.1 ` tun_a的路由表和源IP`180.163.26.39`不匹配。因此需要关闭rp_filter。

```bash
# 关闭rp_filter检查
 echo 0 > /proc/sys/net/ipv4/conf/tun_a/rp_filter
 echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter
 # 或者设置rp_filter为2
 echo 2 > /proc/sys/net/ipv4/conf/tun_a/rp_filter
 echo 2 > /proc/sys/net/ipv4/conf/all/rp_filter
```

当然，除了关闭rp_filter，还有一种更好的方式: 为`qq.com(180.163.26.39)`设置路由：

```bash
ip route add 180.163.26.39 dev tun_a proto kernel scope link src 192.168.1.1
```

设置路由后，在机器A上访问`qq.com(180.163.26.39)`时，根据路由表，协议栈默认会选择走tun_a IPIP隧道发出数据报。因此在可以直接在机器A上执行`ping 180.163.26.39`，而不需要使用`-I`选项指定特定的网络接口。



#### 参考资料

- [iptables 教程](http://www.zsythink.net/archives/category/运维相关/iptables/)

