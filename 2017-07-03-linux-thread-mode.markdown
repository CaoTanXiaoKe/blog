---
layout:     post
title:      "Linux中的线程存储器模型"
subtitle:   "线程存储器模型，线程内存区共享问题"
date:       2017-07-03
author:     "ChenWenKe"
header-img: "img/post_csapp/vm.jpg"
tags:
    - Linux
    - 多线程
---
#### 线程存储器模型
> 一组并发线程运行在一个进程的上下文中。 每个线程都有它自己独立的线程上下文，包括**线程ID, 栈，栈指针，程序计数器，条件码和通用目的寄存器值。** 每个线程和其他线程一起共享进程上下文的剩余部分。这包括整个用户虚拟地址空间，它是有只读文本（代码），读/写数据，堆以及所有的共享库代码和数据区域组成的。线程也共享同样的打开文件的集合。<br/>
从实际操作角度来说，让一个线程去读或写另一个线程的寄存器值是不可能的。 另一方面，任何线程都可以访问共享虚拟存储器的任意位置。如果某一个线程修改了一个存储器位置，那么其他每个线程最终都能在它读这个位置时发现这个变化。 因此，**寄存器是从不共享的，而虚拟存储器总是共享的。**<br/>
各自独立的线程栈的存储器模型不是那么整齐清楚的。这些栈被保存在虚拟地址空间的栈区域中，并且通常是被相应的线程独立地访问的。我们说通常而不是总是，是因为不同的线程栈是不对其他线程设防的。 所以，如果一个线程以某种方式得到一个指向其他线程栈的指针，那么它就可以读写这个栈的任何部分。 

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

void *thread(void *vargp); 
char **ptr;		// Clobal variable

int main(int argc, char** argv)
{
	int i; 
	pthread_t tid; 
	char *msgs[2] = 
	{
		"Hello from foo", 
		"Hello from bar"
	}; 

	ptr = msgs; 
	for(i = 0; i < 2; i++)
		pthread_create(&tid, NULL, thread, (void *)i); 
	pthread_exit(NULL); 
}

void *thread(void *vargp)
{
	int myid = (int)vargp; 
	static int cnt = 0; 	// global/static area
	printf("[%d]: %s (cnt=%d)\n", myid, ptr[myid], ++cnt); 

	int st = 1; 
	printf("st : %d\n", st); 

	const int cst = 5;	// const area 
	printf("const int addr: %llu\n", (unsigned long long)&cst); 

	return NULL; 
}

```

输出：
>
[1]: Hello from bar (cnt=1)<br/>
st : 1<br/>
const int addr: 18446744072484184964<br/>
[0]: Hello from foo (cnt=2)<br/>
st : 1<br/>
const int addr: 18446744072492577668


由上面的程序执行结果可知：全局和静态变量的更改体现在每个线程中，当然动态分配的内存中的内容中的变动也会体现在所有线程中，也就是说它们在当前进程的虚拟地址空间中只有一份实例。原则上来讲，各个线程的栈都是相互独立的。但是由于每个线程都能访问该进程的整个虚拟地址空间，因此只要有另一个线程的栈地址，就能访问。 例如：上面例子中 msgs是存储在主线程的栈中的。 

关于C++中的内存区可以参考这篇译文：[C++中的内存区](https://caotanxiaoke.github.io/CaoTanXiaoKe.github.io/2017/06/07/memory-areas-in-c++/)

---

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <unistd.h>

void *thread(void *vargp); 
char *ptr;		// Clobal variable

int main(int argc, char** argv)
{
	int i; 
	pthread_t tid; 
	
	ptr = malloc(sizeof(char) * 30); 
	strcpy(ptr, "Nothing replaces hard work"); 
	for(i = 0; i < 2; i++)
		pthread_create(&tid, NULL, thread, (void *)i); 
	pthread_exit(NULL); 
}

void *thread(void *vargp)
{
	printf("%s\n", ptr); 
	return NULL; 
}
```

输出：
> Nothing replaces hard work<br/>
Nothing replaces hard work

<br/>

#### 参考资料
- 《深入理解计算机系统 原书2th》(CSAPP) 12章-并发编程

<br/>
<br/>