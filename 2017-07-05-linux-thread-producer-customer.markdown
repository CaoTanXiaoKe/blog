---
layout:     post
title:      "Linux生产者消费者问题"
subtitle:   "Linux线程同步，信号量，生产者消费者问题"
date:       2017-07-05
author:     "ChenWenKe"
header-img: "img/post_csapp/vm.jpg"
tags:
    - Linux
    - 多线程
---

#### 线程同步的方式
- 互斥量
- 读写锁
- 条件变量
- 自旋锁
- 屏障 (barrier)
- 信号量

下文是用信号量实现 **生产者-消费者** 模型。

#### 生产者-消费者问题
生产者-消费者问题。 生产者和消费者线程共享一个有 n 个槽的有限缓冲区。生产者线程反复地生成新的"项目"（item）, 并把它们插入到缓冲区中。消费者线程不断的从缓冲区中取出这些“项目”，然后消费（使用）它们。也可能有多个生产者和消费者的变种。

生产者 —— 消费者的相互作用在现实系统中是很普遍的。 例如，在一个多媒体系统中，生产者编码视频帧，而消费者解码并在屏幕上呈现出来。 缓冲区的目的是为了减少视频流的抖动（jitter）, 而这种抖动是由各个帧的编码和解码时与数据相关的差异引起的。缓冲区为生产者提供了一个槽位池，而为消费者提供了一个已编码的帧池。 另一个常见的示例是图形用户接口设计。 生产者检测到鼠标和键盘事件，并将它们插入到缓冲区中。消费者以某种基于优先级的方式从缓冲区取出这些事件，并显示在屏幕上。 


---


关于代码的简单解释："项目"存放在一个动态分配的 n 项整数数组（buf）中。 front 和 rear 索引值记录该数组中的第一项和最后一项。 三个信号量同步对缓冲区的访问。mutex 信号量提供互斥的缓冲区访问。slots 和 items 信号量分别记录空槽位和可用“项目”的数量。


```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <pthread.h>
#include <semaphore.h>
#include <time.h>

void P(sem_t *s)
{
	sem_wait(s); 
}

void V(sem_t *s)
{
	sem_post(s); 
}

typedef struct 
{
	int *buf;			// Buffer array
	int n; 				// Maximum number of slots
	int front; 			// buf[(front+1)%n] is first item
	int rear;			// buf[(rear%n)] is last item
	sem_t mutex;		// Protects accesses to buf
	sem_t slots; 		// Counts available slots
	sem_t items; 		// Counts available items 
} sbuf_t;


// Create an empty, bounded, shared FIFO buffer with n slots
void sbuf_init(sbuf_t *sp, int n)
{
	sp->buf = malloc(n * sizeof(int)); 
	sp->n = n;						// Buffer holds max of n items
	sp->front = sp->rear = 0;		// Empty buffer iff front = rear
	sem_init(&sp->mutex, 0, 1);		// Binary semaphore for locking
	sem_init(&sp->slots, 0, n);		// Initially, buf has n empty slots
	sem_init(&sp->items, 0, 0);		// Initially, buf has zero data items
}

// Clean up buffer sp
void sbuf_deinit(sbuf_t *sp)
{
	free(sp->buf); 
}

// Insert item onto the rear of shared buffer sp
void sbuf_insert(sbuf_t *sp, int item)
{
	P(&sp->slots);			// Wait for available slot 
	P(&sp->mutex); 			// Lock the buffer
	sp->buf[(++sp->rear)%(sp->n)] = item; 	// Insert the item
	V(&sp->mutex); 			// Unlock the buffer
	V(&sp->items); 			// Announce available item
}

// Remove and return the first item from buffer sp
int sbuf_remove(sbuf_t *sp)
{
	int item; 
	P(&sp->items); 		// Wait for available item
	P(&sp->mutex); 		// Lock the buffer
	item = sp->buf[(++sp->front)%(sp->n)];	// Remove the item
	V(&sp->mutex); 		// Unlock the buffer
	V(&sp->slots); 		// Announce available slot
	return item; 
}

void *creater(void* sbuffer)
{
	pthread_detach(pthread_self()); 	// detach thread
	sbuf_t *sp = (sbuf_t *) sbuffer; 
	int i = 0; 
	while(1)
	{
		usleep((rand()%100) * 10000); 
		// producer items ...
		i++; 	
		sbuf_insert(sp, i); 
		printf("creater put %d into sbuffer\n", i); 
	}
}

void *customer(void* sbuffer)
{
	pthread_detach(pthread_self()); 
	sbuf_t *sp = (sbuf_t *) sbuffer; 
	int item; 
	while(1)
	{
		usleep((rand()% 100) * 10000);
		item = sbuf_remove(sp); 
		printf("customer get %d from sbuffer\n", item);
		// cutomer items ... 
	}
}

int main()
{
	sbuf_t sbuffer; 
	sbuf_init(&sbuffer, 20); 
	
	srand(time(NULL)); 

	pthread_t tid1, tid2; 
	pthread_create(&tid1, NULL, creater, &sbuffer); 
	pthread_create(&tid2, NULL, customer, &sbuffer); 

	sbuf_deinit(&sbuffer); 
	pthread_exit(0); 		// Don't use exit, return,,,
}
```

关于上面代码中的 `pthread_detach` 和 `pthread_exit` 可以看[这里](https://caotanxiaoke.github.io/CaoTanXiaoKe.github.io/2017/07/04/linux-thread_join-thread_detach/)。

<br/>
#### 参考资料
- 《深入理解计算机系统 原书2th》(CSAPP) 12章-并发编程

<br/>
<br/>