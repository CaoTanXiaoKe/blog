---
layout:     post
title:      "IPIP协议实例分析《一》"
subtitle:   "配置ipip实例，分析ipip协议封包，解包，路由的过程"
date:       2019-08-16
author:     "ChenWenKe"
tags:
	- TCP/IP
	- 网络编程
mathjax: true
---

​        IPIP见文生义，即IP报文外面又封装了一层IP报文。内层的IP报文为原报文，外层报文的源IP和目的IP，即是隧道两端的IP。也就是说，应用想要通过隧道在源IP报文外再封装一层IP报文，从而使得原报文能够通过隧道路由到对端。对端的IPIP隧道模块，把原来的外层IP拆解掉，从而还原出原来的IP报文。

​		也就是说，IPIP主要做的就是封包，解包的过程。通过IPIP封解包，原始IP报文能够在外层IP的托运下，通过路由穿过隧道，抵达隧道的对端，并且还原出原来的IP报文。这就是IPIP隧道的核心思想。

​		IPIP隧道技术具有非常实用的价值，例如打通两个互不冲突的私有网络，在IPIP隧道基础上做VPN。Linux里已经实现了IPIP，下面以几个实例具体看一下IPIP隧道的工作原理。

```
                                         +---------------------------+
                                         |                           |
                                         |      Outer IP Header      |
                                         |                           |
     +---------------------------+       +---------------------------+
     |                           |       |                           |
     |         IP Header         |       |         IP Header         |
     |                           |       |                           |
     +---------------------------+ ====> +---------------------------+
     |                           |       |                           |
     |                           |       |                           |
     |         IP Payload        |       |         IP Payload        |
     |                           |       |                           |
     |                           |       |                           |
     +---------------------------+       +---------------------------+
```



### 准备

- 在同一地域下的购买两台云主机(腾讯云, CentOS 7.6 64位)

- 机器A： eth0 IP : `172.16.16.22`

- 机器B:  eth0 IP : `172.16.1.62`

  **注意:** 上面两台机器的eth0需要是互通的, 即同一个VPC。

​        有了互通的IP之后，我们可以把这对互通的IP分别做为隧道的两端。我们还需要分别位于机器A，机器B不互通的IP作为隧道托运的对象。

​        因此我们打算从`192.168.0.0/16`私有IP网段上选择两个IP(`192.168.1.1`, `192.168.1.254`)，分别赋予机器A，机器B。为方便起见，我们直接把IP赋予隧道上tun设备上。



### 启动IPIP模块

- 检查ipip模块和tun模块是否已经启动

  ```bash
  lsmod | grep ipip
  lsmod | grep tun
  ```

- 如果ipip模块，tun模块没有启动，分别启动它们。

  ```bash
  modprobe ipip
  modprobe tun
  ```

  如果启动成功，可以观察到：

  ```
  [root@VM_1_62_centos ~]# lsmod | grep ipip
  ipip                   13465  0
  tunnel4                13252  1 ipip
  ip_tunnel              25163  1 ipip
  [root@VM_1_62_centos ~]#
  [root@VM_1_62_centos ~]# lsmod | grep tun
  tun                    31740  0
  tunnel4                13252  1 ipip
  ip_tunnel              25163  1 ipip
  [root@VM_1_62_centos ~]#
  ```



### 实例一：

#### 配置隧道并测试

- 在机器A上创建隧道：

  ```bash
  # 测试机器A和机器B的eth0是否能够互相访问
  ping 172.16.1.62 -c 1
  # 创建隧道 tun_a
  ip tunnel add tun_a mode ipip remote 172.16.1.62 local 172.16.16.22
  # 开启隧道 tun_a
  ip link set tun_a up
  # 查看
  ip a
  ```

- 在机器B上创建隧道：

  ```bash
  # 创建隧道 tun_b
  ip tunnel add tun_b mode ipip remote 172.16.16.22 local 172.16.1.62
  # 开启隧道 tun_b
  ip link set tun_b up
  # 查看
  ip a
  ```

- 为机器A的隧道配置私有IP: `192.168.1.1`

  ```bash
  ip addr add 192.168.1.1/24 dev tun_a
  ```

- 为机器B的隧道配置私有IP: `192.168.1.254`

  ```
  ip addr add 192.168.1.254/24 dev tun_b
  ```

- 在机器A上执行下面命令，测试隧道是否连通:

  ```bash
  [root@VM_16_22_centos ]# ping 192.168.1.254
  PING 192.168.1.254 (192.168.1.254) 56(84) bytes of data.
  64 bytes from 192.168.1.254: icmp_seq=1 ttl=64 time=1.67 ms
  64 bytes from 192.168.1.254: icmp_seq=2 ttl=64 time=1.67 ms
  ```

- 在机器B上执行下面命令，测试隧道是否连通:

  ```bash
  [root@VM_1_62_centos ~]# ping 192.168.1.1
  PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
  64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=1.63 ms
  64 bytes from 192.168.1.1: icmp_seq=2 ttl=64 time=1.68 ms
  ```

#### 抓包分析

我们以在机器A上执行`ping 192.168.1.254`的过程为例。

- ping进程构造ICMP echo请求包(目的地址为 192.168.1.254)，并通过socket发给协议栈。
- 协议栈根据目的地址和系统路由表，知道目的地址为192.168.1.254的数据包需要发送给tun_a, 并且选择tun_a上的地址192.168.1.1为源地址。
- tun_a隧道对进来的IP数据包封装一层外层IP头，此时数据报变为:  Outer IP Header的IP：`172.16.16.22 --> 172.16.1.62`； Inner IP Header的IP：`192.168.1.1 --> 192.168.1.254`。tun_a把封装后的ipip报文发给协议栈。
- 协议栈根据目的地址和系统路由表，知道目的地址为172.16.1.62的数据包，需要通过eth0发送出去。

​       

上面提到协议栈根据目的地址和系统路由表，决定把数据包通过哪个设备发送到哪里，查看系统路由表如下：

```bash
[root@VM_16_22_centos]# ip route
default via 172.16.16.1 dev eth0
169.254.0.0/16 dev eth0 scope link metric 1002
172.16.16.0/20 dev eth0 proto kernel scope link src 172.16.16.22
192.168.1.0/24 dev tun_a proto kernel scope link src 192.168.1.1
```

​	   

到此，机器A上的发包过程就大致完成了。我们抓包验证一下整个发包过程：

- 抓经过tun_a的包

  ```bash
  [root@VM_16_22_centos]# tcpdump -i tun_a -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on tun_a, link-type RAW (Raw IP), capture size 262144 bytes
  12:34:14.671638 IP 192.168.1.1 > 192.168.1.254: ICMP echo request, id 10606, seq 215, length 64
  12:34:14.673281 IP 192.168.1.254 > 192.168.1.1: ICMP echo reply, id 10606, seq 215, length 64
  12:34:15.673430 IP 192.168.1.1 > 192.168.1.254: ICMP echo request, id 10606, seq 216, length 64
  12:34:15.675059 IP 192.168.1.254 > 192.168.1.1: ICMP echo reply, id 10606, seq 216, length 64
  ```

  如图，经过tun_a的的确是`IP 192.168.1.1 > 192.168.1.254` 的 ICMP echo 请求包。

- 抓经过eth0的包

  ```bash
  [root@VM_16_22_centos]# tcpdump -i eth0 not tcp -nnn
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
  12:37:04.951909 IP 172.16.16.22 > 172.16.1.62: IP 192.168.1.1 > 192.168.1.254: ICMP echo request, id 10606, seq 385, length 64 (ipip-proto-4)
  12:37:04.953513 IP 172.16.1.62 > 172.16.16.22: IP 192.168.1.254 > 192.168.1.1: ICMP echo reply, id 10606, seq 385, length 64 (ipip-proto-4)
  ```

  从eth0的数据包上可以看出，数据包经过eth0发出去时，的确是`IP 172.16.16.22 > 172.16.1.62: IP 192.168.1.1 > 192.168.1.254`的ICMP echo请求包。



到此，我们看一下机器A的ICMP请求包通过隧道到达机器B后，机器B上的处理过程:

- 首先机器B的eth0接收到`IP 172.16.16.22 > 172.16.1.62: IP 192.168.1.1 > 192.168.1.254`的ICMP echo请求包。根据目的地址，发现是到达该机器的数据包，把数据包交给协议栈。
- 协议栈解析数据包时发现，这是一个ipip的数据包，解包得到内层数据包`192.168.1.1 > 192.168.1.254`的ICMP 请求包，目的地址是`192.168.1.254` 是tun_b的ip， 构造ICMP的应答报文：`IP 192.168.1.254 > 192.168.1.1: ICMP echo reply`。
- 目的地址是`192.168.1.1`， 因此协议栈根据系统路由把应答数据包发送给tun_b。
- tun_b隧道对进来的IP数据包封装一层外层IP头，此时数据报变为:  Outer IP Header的IP：`172.16.1.62 > 172.16.16.22`； Inner IP Header的IP：`192.168.1.254 > 192.168.1.1`。tun_b把封装后的ipip报文发给协议栈。
- 协议栈根据目的地址和系统路由表，知道目的地址为172.16.16.22的数据包，需要通过eth0发送出去。

然后机器A从eth0上接收到IPIP的回包，把回包提交给协议栈，协议栈对数据包进行解析，转交给tun_a设备，tun_a设备把数据包内容提交给协议栈，查找相关套接字，拷贝数据包内容到套接字缓冲区，最终回显ICMP 请求的应答消息。

- 在机器B上分别对tun_b和eth0进行抓包：

```bash
[root@VM_1_62_centos ~]# tcpdump -i tun_b -nnn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tun_b, link-type RAW (Raw IP), capture size 262144 bytes
12:52:13.427532 IP 192.168.1.1 > 192.168.1.254: ICMP echo request, id 10606, seq 1292, length 64
12:52:13.427559 IP 192.168.1.254 > 192.168.1.1: ICMP echo reply, id 10606, seq 1292, length 64
12:52:14.429296 IP 192.168.1.1 > 192.168.1.254: ICMP echo request, id 10606, seq 1293, length 64
12:52:14.429318 IP 192.168.1.254 > 192.168.1.1: ICMP echo reply, id 10606, seq 1293, length 64
```

```bash
[root@VM_1_62_centos ~]# tcpdump -i eth0 not tcp -nnn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
13:05:01.644622 IP 172.16.16.22 > 172.16.1.62: IP 192.168.1.1 > 192.168.1.254: ICMP echo request, id 10606, seq 2059, length 64 (ipip-proto-4)
13:05:01.644680 IP 172.16.1.62 > 172.16.16.22: IP 192.168.1.254 > 192.168.1.1: ICMP echo reply, id 10606, seq 2059, length 64 (ipip-proto-4)
```



至此，整个一台机器连接的私有网络通过ipip访问另一台机器连接的私有网络的大致流程就结束了。 上面的整体过程中，一些细节我还没经过确认，可能存在谬误，但大致的概念模型应该偏差不大，欢迎交流学习。



### 参考资料

- [ IP Encapsulation within IP](https://tools.ietf.org/html/rfc2003)
- [How to establish IPIP tunnel on Linux](https://sites.google.com/site/suminknowledgebase/linux-knowledge-base/howtoestablishipiptunnelonlinux)
- [Linux虚拟网络设备之tun/tap](https://segmentfault.com/a/1190000009249039)
- [Linux网络 - 数据包的接收过程](https://segmentfault.com/a/1190000008836467)