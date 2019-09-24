---
layout:     post
title:      "Raft协议解读"
subtitle:   "对Raft协议的论文进行简单解读"
date:       2019-07-15
author:     "ChenWenKe"
tags:
	- 算法
    - 分布式
    - 共识
mathjax: true
---

Raft协议是目前使用非常广泛的分布式共识协议， 它有和paxos协议相当的性能和容错能力，但是相对而言却更简单清晰，易于工程实现。Raft协议把协议主要分成Leader选举，日志复制，安全性约束等模块，以便于理解和实现。

## Raft协议论文

 [In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)

​
(哪里有什么捷径，论文还是需要反反复复的读)


### raft协议架构

![节点状态机](/Users/wikichen/Desktop/MyProject/blog/img/raft-arch.png)

每一个节点都实现了一个状态机，日志模块和一致性模块。它们的功能分别是：

- 状态机：数据一致性就是指的状态机的一致性，从内部服务的角度看来，状态机中的数据都保持一致。
- log模块：保存了所有的操作记录。
- 一致性模块：一致性模块是raft协议的核心，保证了写入log的记录的一致性。

## 选主

#### 竞选过程

- 节点按照一定的随机时间范围，由follower变成candidate，同时设置当前的Term。
- candidate给自己投票，并且向其他节点发送带有Term和LogIndex的请求，征集其他节点的选票。
- 等待选举结果，如果当选成为Leader，如果接到Leader的心跳(有其它节点当选了)成为follower，否则随机等待一段时间，再次发起拉票请求。

#### 选举结果

- 成功当选：

  收到超过半数的选票时，节点成为leader，后续定时向其它节点发送心跳，并携带上Term信息，如果其它节点发现自己的当前Term小于Leader发送的Term，将自己的状态置为follower。

- 选举失败：

  在candidate状态时，如果收到其它节点(leader)发送过来的心跳信息，并且Term大于自己的Term，状态转化为follower。

- 未产生结果：

  没有一个canditate获取到超过一半的选票，canditate随机退避一段时间进入下一轮的竞选。



## 日志拷贝

![日志提交](/Users/wikichen/Desktop/MyProject/blog/img/log-replication.png)

日志复制是raft算法功能模块的核心，是读论文时最容易产生误解和疑惑的地方。日志复制过程如下：

1. 客户端发送请求给leader或者follower，follower接到客户端的请求会把请求转发给leader，leader接到请求后把请求日志追加到本地日志(未应用和提交)
2. leader把日志复制给所有follower节点，follower节点接到复制日志的请求后，进行检验(需要满足安全性限制)，如果要复制的日志经过了安全性检验，把复制日志追加到本地日志。
3. 如果收到超过半数的节点(包括leader本身)返回复制日志成功。 在状态机中应用该日志，并且进行提交。
4. leader广播所有follower节点进行执行状态机应用日志和提交日志。
5. 如果超过一半的节点成功提交日志，leader给客户端返回请求成功。
=======
占坑

## Raft协议概述

## Raft协议论文

## 选主

## 日志拷贝

## 安全性

## 节点变更

## 快照
