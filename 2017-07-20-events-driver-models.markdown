---
layout:     post
title:      "select /pselect / poll / epoll"
subtitle:   "对 select, pselect, poll, epoll函数的简要概况，以及select跟epoll的比较"
date:       2017-07-20
author:     "ChenWenKe"
header-img: "img/post_csapp/vm.jpg"
tags:
    - Linux
    - OS
    - 网络编程
    - 多线程
---

#### select /pselect / poll / epoll

#### 预备知识：
TCP套接字编程， [阻塞I/O与非阻塞I/O](/2017/07/16/IO-models/)

### 各个函数简介

#### select 函数: 
- 该函数允许进程指示内核等待多个事件中的任何一个发生，并只有在有一个或多个事件发生或经历一段指定的时间后才唤醒它。
- **调用select函数时的注意事项：** maxfdp1参数（第一个参数）指定待测试的描述符个数，如 maxfdp1为 5，则描述符：0,1,2,3,,,maxfdp1均可被测试。 函数返回的值指明了准备好集合的基数。并且准备好集合是**值-结果**方式传递和返回的，因此我们每次调用 select函数时，我们都得再次把所有描述符集合内所关心的位置置为 1. 
- 注意： 当某个套接字上发生错误时，它将由 select标记为即可读又可写的。 

#### pselect 函数：
pselect和select提供的功能类似，但pselect 能够处理信号阻塞并提供了更高时间分辨率。

#### poll 函数：
poll 函数和select提供的功能也类似，不过在处理流设备时，它能提供额外的信息（优先级信息）。 

#### epoll 函数：
epoll 提供的功能也同样和select类似, 但是 epoll更加高效，并且支持两种触发方式：水平触发和边缘触发。 

### select VS epoll

- 要比较epoll相比较select高效在什么地方，就需要比较二者做相同事情的方法。要完成对I/O流的复用需要完成如下几个事情：
	1. 用户态怎么将文件描述符传递到内核态？
	2. 内核态怎么判断I/O流可读可写？
	3. 内核怎么通知监控者有I/O流可读可写？
	4. 监控者如何找到可读可写的I/O流并传递给用户态应用程序？
	5. 继续循环时监控者怎样重复上述步骤？搞清楚上述的步骤也就能解开epoll高效的原因了。

- select的做法：
	1. 步骤1的解法：select创建3个文件描述符集，并将这些文件描述符拷贝到内核中，这里限制了文件描述符的最大的数量为1024（注意是全部传入---第一次拷贝）；
	2. 步骤2的解法：内核针对读缓冲区和写缓冲区来判断是否可读可写,这个动作和select无关；
	3. 步骤3的解法：内核在检测到文件描述符可读/可写时就产生中断通知监控者select，select被内核触发之后，就返回可读可写的文件描述符的总数；
	4. 步骤4的解法：select会将之前传递给内核的文件描述符再次从内核传到用户态（第2次拷贝），select返回给用户态的只是可读可写的文件描述符总数，再使用FD_ISSET宏函数来检测哪些文件I/O可读可写（遍历）；
	5. 步骤5的解法：select对于事件的监控是建立在内核的修改之上的，也就是说经过一次监控之后，内核会修改位，因此再次监控时需要再次从用户态向内核态进行拷贝（第N次拷贝）

- epoll的做法：
	1. 步骤1的解法：首先执行epoll_create在内核专属于epoll的高速cache区，并在该缓冲区建立红黑树和就绪链表，用户态传入的文件描述符将被放到红黑树中（第一次拷贝）。
	2. 步骤2的解法：内核针对读缓冲区和写缓冲区来判断是否可读可写，这个动作与epoll无关；
	3. 步骤3的解法：epoll_ctl执行add动作时除了将文件句柄放到红黑树上之外，还向内核注册了该文件描述符的回调函数，内核在检测到某文件描述符可读可写时则调用该回调函数，回调函数将文件描述符放到就绪链表。
	4. 步骤4的解法：epoll_wait只监控就绪链表就可以，如果就绪链表有文件描述符，则表示该文件描述符可读可写，并返回到用户态（少量的拷贝）；
	5. 步骤5的解法：由于内核不修改文件描述符的位，因此只需要在第一次传入就可以重复监控，直到使用epoll_ctl删除，否则不需要重新传入，因此无多次拷贝。

简单说：epoll是继承了select/poll的I/O复用的思想，并在二者的基础上从监控IO流、查找I/O事件等角度来提高效率，具体地说就是内核文件描述符列表、红黑树、就绪list链表来实现的。

#### epoll综合的执行过程:
一棵红黑树，一张准备就绪文件描述符链表，少量的内核cache，就帮我们解决了大并发下的socket处理问题。执行epoll_create时，创建了红黑树和就绪链表，执行epoll_ctl时，如果增加socket描述符，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据。执行epoll_wait时立刻返回准备就绪链表里的数据即可。

#### epoll水平触发和边缘触发的实现：     
当一个socket文件描述符上有事件时，内核会把该句柄插入上面所说的准备就绪list链表，这时我们调用epoll_wait，会把准备就绪的socket拷贝到用户态内存，然后清空准备就绪list链表， 最后，epoll_wait干了件事，就是检查这些socket，如果不是ET模式（就是LT模式的描述符了），并且这些socket上确实有未处理的事件时，又把该句柄放回到刚刚清空的准备就绪链表了，所以，非ET的句柄，只要它上面还有事件，epoll_wait每次都会返回。而ET模式的描述符，除非有新中断到，即使socket上的事件没有处理完，也是不会次次从epoll_wait返回的。====>区别就在于epoll_wait将socket返回到用户态时是否清空就绪链表。

#### 第三部分：epoll高效的本质

1. 减少用户态和内核态之间的文件描述符拷贝；

2. 减少对可读可写文件描述符的遍历； 

#### 实例代码
- [select 实现简单的回射服务器](http://www.cnblogs.com/acm1314/p/7050324.html)
- [epoll 实现简单的回射服务器](https://banu.com/blog/2/how-to-use-epoll-a-complete-example-in-c/)
- `man 7 epoll`

#### 参考资料
- 《Unix环境高级编程 3th》
- [why is epoll faster than select](https://stackoverflow.com/questions/17355593/why-is-epoll-faster-than-select)
- [epoll 比 select 高效的原因](https://www.zhihu.com/question/20122137/answer/146866418)
- [how to use epoll a complete example in c](https://banu.com/blog/2/how-to-use-epoll-a-complete-example-in-c/)