---
layout:     post
title:      "Linux中thread_join函数thread_detach函数详解"
subtitle:   "线程终止的方式， thread_join函数， thread_detach函数"
date:       2017-07-04
author:     "ChenWenKe"
header-img: "img/post_csapp/vm.jpg"
tags:
    - Linux
    - 多线程
---

#### 线程终止的方式
- 线程所属的进程终止： 如果该进程中的任意线程调用了 [`exit`, `_Exit` 或者 `_exit`](), 那么整个进程就会终止。 与此相类似，如果某信号的默认动作是终止进程，那么发送到线程的该信号就会终止整个进程。
- 线程从启动例程返回，返回值是线程的退出码。
- 线程被同一个进程中的其他线程取消。 
- 线程调用 pthread_exit 函数， 线程会显式地终止。如果主线程调用 pthread_exit, 它会等待所有其他对等线程终止，然后再终止终止主线程和整个进程。 

#### 回收已终止线程的资源

##### pthread_join函数

线程通过调用 pthread_join 函数等待其他线程终止。 

```c
#include <pthread.h>
int pthread_join(pthread_t tid, void **thread_return); 
// 返回： 若成功则返回 0， 若出错则为非 0. 
```
**pthread_join函数会阻塞**，直到线程 tid终止， 将线程例程返回的（void*）指针赋值为 thread_return 指向的位置，然后回收已终止线程占用的所有存储器资源。 跟Unix等待回收进程的函数（wait函数）不同的是，pthread_join函数只能等待一个指定的线程终止。 

##### pthread_detach函数
在任何一个时间点上，线程是**可结合(joinable)**的或者是**分离的(detached)**。一个可结合的线程能够被其他线程回收其资源和杀死。在被其他线程回收之前，它的存储器资源（例如栈）是没有被释放的。相反，一个分离的线程是不能被其他线程回收或杀死的。它的存储器资源在它终止时由系统自动释放。 
默认情况下，线程被创建成可结合的。 为了避免存储器泄露，每个可结合线程都应该要么被其他线程显式地收回，要么通过调用 pthread_detach函数被分离。 

```c
#include <pthread.h>
int pthread_detach(pthread_t tid); 
// 返回：若成功则返回0， 若出错则为非零。 
```

###### pthread_join函数和pthread_detach函数的区别

在默认情况下，线程的终止状态会保存直到对该线程调用 pthread_join。如果线程已经被分离，线程的底层存储资源可以在线程终止时立即被系统收回。在线程被分离后，我们不能用pthread_join函数等待它的终止状态，因为对分离状态的线程调用 pthread_join会产生未定义行为。

stackoverflow上有人是这样描述的：
>pthread_detach just means that you are never going to join with the thread again. This allows the pthread library to know whether it can immediately dispose of the thread resources once the thread exits (the detached case) or whether it must keep them around because you may later call pthread_join on the thread.
Once main returns (or exits) the OS will reap all your threads and destroy your process.

斗胆翻译一下，大概是这个意思：
> pthread_detach 意味着，线程一旦用pthread_detach分离，就不要再用pthread_join结合了（因为这样做是未定义的行为）。 pthread_detach分离线程就是告诉 pthread library 一旦线程结束，可以立即回收该线程所拥有的资源。但如果是结合线程，则必须等到调用pthread_join才能回收线程资源。 
如果 主线程（其实可以是线程组中的任意一个线程）return, exit, _Exit, _exit。操作系统会终止进程，并回收该进程中线程组中的所有线程。


#### 实例

```c
// pthread_join() 实例
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void *thread(void *vargp); 

int main(int argc, char **argv)
{
	pthread_t tid; 
	char *ptr = "Nothing replaces hard work"; 
	pthread_create(&tid, NULL, thread, ptr); 
	pthread_join(tid, NULL); 

	pthread_exit(0); 
}

void *thread(void *vargp)
{
	char *ptr = (char *)vargp; 
	printf("%s\n", ptr); 
	sleep(5); 
	return NULL; 
}
```

<br/>

```c
// pthread_detach() 实例
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void *thread(void *vargp); 

int main(int argc, char **argv)
{
	pthread_t tid; 
	char *ptr = "Nothing replaces hard work"; 
	pthread_create(&tid, NULL, thread, ptr); 

	pthread_exit(0); 	// 注意，这里不要用 exit(3): return, exit, _Exit, _exit
}

void *thread(void *vargp)
{
	pthread_detach(pthread_self()); 
	char *ptr = (char *)vargp; 
	printf("%s\n", ptr); 
	sleep(5); 
	return NULL; 
}
```

<br/>
#### 参考资料
- 《深入理解计算机系统 原书2th》(CSAPP) 12章-并发编程
- 《Unix环境高级编程 3th》（APUE）11章-线程

<br/>
<br/>