---
layout:     post
title:      "Goroutine 调度"
subtitle:   "Goroutine的实现与调度机制"
date:       2019-12-20
author:     "ChenWenKe"
tags:

  - golang
  - OS
  - 并发
mathjax: true

---


> 好的设计一定是在精通的基础上，而不是从无知中产生的。  
>
>  — 《设计原本》Frederick P. Brooks

# 协程

## 协程是什么

![img](/blog/img/goroutine_scheduler/case1.png)

我们说**协程是用户态的线程**。这样就牵涉到，线程是什么，进程是什么，以及它们和协程的区别。

- 进程：由内核来调度和维护，因为进程有独立的虚拟地址空间，想要和其他进程（逻辑流）通信，控制流必须使用某种显式的进程间通信（interprocess communication, IPC）机制。

  - 独立的虚拟地址空间。

  - 独立的打开文件描述符表。

  - 由内核进行调度。

- I/O 多路复用：在这种形式的并发编程中，应用程序在一个进程的上下文中显式地调度他们自己的逻辑流。逻辑流被模型化为状态机，数据到达文件描述符后，主程序显式地从一个状态转换到另一个状态。因为程序是一个单独的进程，所以所有的流都共享同一个地址空间。 最典型的例子： Nginx 结合使用了多进程和I/O多路复用的技术。

- 线程: 线程是运行在一个单一进程上下文中的逻辑流，由内核进行调度。每个线程都有它自己的上下文。典型案例：apache 。

  - 共享地址空间，原则上一个进程内的线程只有寄存器里的值不能相互访问。

  - 由内核进行调度。
  - 包含唯一的线程ID，栈，栈指针，程序计数器，通用目的寄存器和条件码。

- 协程 : 在用户态抽象出的逻辑流，在用户态维护该逻辑流的上下文，以模拟函数调用的方式切换上下文。可以简单把协程看做是对I/O多路复用的抽象。

  - 共享地址空间，类似于用户态函数调用。

  - 用户态代码进程调度（例如golang的runtime，libco协程库本身）。

## 为什么需要协程:

![img](/blog/img/goroutine_scheduler/the_i7_memory_system.png)

> 唯一不变的，只有变化。

### 我们目前的开发现状：今非昔比

- 多核心,存储器体系结构:我们需要充分利用多核的能力，我们的程序需要更好的局部性。

- 虚拟机，容器化：通常一个虚拟机或容器，主要运行一个服务。

- Web应用的特点：IO密集，高并发。

- 多进程，多线程，I/O多路复用的缺点与挑战。

  - 多进程：重。

  - 多线程：重。

  - I/O多路复用： 复杂度太高。

### goroutine设计的目标

- 可伸缩性(scability)

- 简单(minimal API)

- 调度公平

- 无尽栈空间(infinite stack)

## 实现协程的基本原理

### 函数调用的过程

![img](/blog/img/goroutine_scheduler/stack_frame_structure.png)

```assembly
// C source code
int add(int a, int b)
{
  int c = a + b;
  return c;
}
int main()
{
  int a, b, sum;
  a = 2;
  b = 3;
  sum = add(a, b);
  return 0;
}
# Intermix source code with disassembly
0000000000000660 :
int add(int a, int b)
{
 660:     55              push  %rbp
 661:     48 89 e5           mov   %rsp,%rbp
 664:     89 7d ec           mov   %edi,-0x14(%rbp)
 667:     89 75 e8           mov   %esi,-0x18(%rbp)
    int c = a + b;
 66a:     8b 55 ec           mov   -0x14(%rbp),%edx
 66d:     8b 45 e8           mov   -0x18(%rbp),%eax
 670:     01 d0             add   %edx,%eax
 672:     89 45 fc           mov   %eax,-0x4(%rbp)
    return c;
 675:     8b 45 fc           mov   -0x4(%rbp),%eax
}
 678:     5d              pop   %rbp
 679:     c3              retq
000000000000067a :
int main()
{
 67a:     55              push  %rbp
 67b:     48 89 e5           mov   %rsp,%rbp
 67e:     48 83 ec 10          sub   $0x10,%rsp
    int a, b, sum;
    a = 2;
 682:     c7 45 fc 02 00 00 00     movl  $0x2,-0x4(%rbp)
    b = 3;
 689:     c7 45 f8 03 00 00 00     movl  $0x3,-0x8(%rbp)
    sum = add(a, b);
 690:     8b 55 f8           mov   -0x8(%rbp),%edx
 693:     8b 45 fc           mov   -0x4(%rbp),%eax
 696:     89 d6             mov   %edx,%esi
 698:     89 c7             mov   %eax,%edi
 69a:     e8 c1 ff ff ff        callq  660 
 69f:     89 45 f4           mov   %eax,-0xc(%rbp)
    return 0;
 6a2:     b8 00 00 00 00        mov   $0x0,%eax
}
 6a7:     c9              leaveq
 6a8:     c3              retq
 6a9:     0f 1f 80 00 00 00 00     nopl  0x0(%rax)
```

相关细节可以参考资料[《深入理解计算机系统》2th，第三章：程序的机器级表示](https://piazza.com/class_profile/get_resource/j7ly9riuca97on/ja86xbbpp0b73b)。

### 简单的例子： libco协程上下文切换

```assembly
# leal(Load Effective Address)，保存 %esp+4 到 %eax
# 可以认为保存当前协程栈sp的位置。
leal 4(%esp), %eax //sp
# 按照函数调用从右到左入栈的顺序，%esp指向了第一个参数: 当前协程 coctx_t 的位置
movl 4(%esp), %esp
# 使得 %esp 指向 regs[] 数组，方便后续使用pushl保存寄存器的值。
leal 32(%esp), %esp //parm a : &regs[7] + sizeof(void)
# 保存各寄存器的值。
pushl %eax //esp ->parm a
pushl %ebp
pushl %esi
pushl %edi
pushl %edx
pushl %ecx
pushl %ebx
# -4(%eax)是刚进入该函数时，%esp指向的位置，即call压入的函数返回地址(PC寄存器的值)
# 仍旧放到当前协程栈的栈顶，用作当前协程再次被调度时，ret跳转到原来地址的下一条指令。
pushl -4(%eax)
# %eax + 4 指向第二个参数，即目的协程的 coctx_t
movl 4(%eax), %esp //parm b -> &regs[0]
# 恢复各寄存器的值
popl %eax  //ret func addr
popl %ebx  
popl %ecx
popl %edx
popl %edi
popl %esi
popl %ebp
popl %esp
pushl %eax //set ret func addr, 当前%esp指向该位置。
# 清零 %eax
xorl %eax, %eax
# 跳转到目的协程的PC地址，开始执行目的协程。
ret     
```

具体细节可以参考[腾讯C++协程库：Libco](https://github.com/Tencent/libco)

## goroutine的调度策略

![img](/blog/img/goroutine_scheduler/gmp1.png)

 ![img](/blog/img/goroutine_scheduler/gmp2.png)

### GMP的概念

- G： 表示Goroutine，可以认为是用户空间内的线程，是一段指令的逻辑流。每一个Goroutine都包含有堆，栈，指令指针(指向该协程执行到哪儿了)等信息。

- M: machine，表示操作系统线程，可以近似的认为就是POSIX线程。

- P: 表示调度器的上下文，默认和机器的逻辑核心数相同，每个P都带有一个装载G的一个本地队列，malloc缓存，栈缓存等。原则上说，一个可运行的G只有被一个空闲的P调度时，才能够运行；golang调度器通过控制P的数量和逻辑核心数相同，巧妙的实现了CPU核心的亲和性，局部性，还避免了M数量的膨胀。

### 可伸缩(Scalability)

为了支持创建大量的协程，充分利用CPU的能力。我们需要在调度协程的时候尽可能避免锁的争抢，尽可能的具有局部性，对于系统调用，C 函数调用，网络IO等阻塞操作发生时，最好能让出CPU等资源。

#### Local Queue 和 Global Queue

![img](/blog/img/goroutine_scheduler/local_queue_global_queue.png)

每一个P都有一个装载G的 Local Queue, 当这个P空闲时倾向于优先从这个本地队列取待运行的协程进行执行，由于每个P都有自己的Local Queue因此减少了锁的争抢。由于每个P一般对应一个逻辑核心，Local Queue上的协程一般情况下都在一个P上进行执行，因此提高了指令执行的局部性，充分利用了存储器缓存。

#### 协程栈

为了支持上百万的协程数量，协程必须足够轻量。协程特有的信息一般都存储在协程栈里，因此很大程度上协程栈的大小决定了协程的规模。goroutine的协程栈是以一种动态增长，按需翻倍增长的方式进行分配的。初始时，栈大小为2k；每个函数调用开始时，都会执行编译器插入的检查栈大小是否溢出的代码，如果溢出，则分配两倍的内存…

```c
// The minimum size of stack used by Go code
_StackMin = 2048
```

#### 系统调用

​                                                 ![img](/blog/img/goroutine_scheduler/syscall.png)

众所周知，进行系统调用的时候，需要进行用户上下文和内核上下文的切换，一旦切入内核上下文，整个线程的调度就由内核进行接管了。有些比较**重**而且不是一直抢占着CPU的操作（例如:SYS_EPOLL_WAIT，SYS_MMAP2, SYS_FSTAT64, SYS_SYNC等）需要在执行到内核的上下文切换之前，让出P(调用entersyscall)。在从系统调用返回后，需要执行等待被P调度的代码(调用exitsyscall)。而一些比较轻量的系统调用(例如：SYS_UMASK， SYS_KILL， SYS_GETTID， SYS_GETPID， SYS_EPOLL_CTL等)，则像普通goroutine一样进行调度，避免执行entersyscall， exitsyscall的额外开销。

#### C Call

执行C call函数调用，golang调度器无法在C函数开始处插入一些检查代码，为了C函数可能会长时间占用P，从而饿死其他就绪的协程。调用C函数也和系统调用做类似的处理。

#### 网络IO

通常情况下网络IO会有很多，而且网络IO相对于本地系统调用，通常会更慢。因此go调度器里维护了一个netpoller, 也就是一个异步的事件循环，统一管理网络IO。这样就避免了像普通的系统调用那样，每个IO操作需要占用一个M（thread）。但是对于netpoller检查网络IO需要调用系统调用，例如epoll_wait, 跟检查等待queue相比开销相对比较大，因此启动另外的线程(sysmon)去间歇性检查网络IO。

- 把所有的网络读写操作设置成非阻塞

- 在读写相关fd时，调用netpoller(在Linux下，把相关fd添加进epoll_event)，如果如果数据未就绪，协程让出P，M，进入等待状态。

- netpoller接受到数据后，唤醒在相应的fd上等待的G，把G设置成可运行状态，放入等待queue。

- 总的来说，对于网络IO，是有专门的后台线程去做事件循环的。**和处理网络IO这种机制相类似的还有： 在Mutexes, Timers, Channels上的阻塞，都有后台线程去轮询。 **

### 公平

```c
runtime.schedule() {
  // only 1/61 of the time, check the global runnable queue for a G.
  // if not found, check the local queue.
  // if not found,
  //   try to steal from other Ps.
  //   if not, check the global runnable queue.
  //   if not found, poll network.
}  
```

#### Local Run Queue 上G如何进行调度

- 整体上是先进先出，公平的调度就绪的协程。

- 但是为了提高缓存命中，应对client-server模型，在FIFO队列前面有一个长度为1的LIFO队列。

#### 如何检查Global Run Queue

当P空闲时，首先会以 1/61的概率去检查Global Run Queue，从而避免Globale Run Queue被饿死。

- 为什么是61？¯\_(ツ)_/¯

  - not too small

  - not too large

  - prime to break any patterns ？？？

#### 抢占

**inherit time slice** -> looks like infinite loop -> preemption (~10ms) 

Goroutine在进行网络IO，阻塞于channel, mutex, timer之前，会让出P和M，然后把自己置为等待状态，放入相应的等待队列上，使得调度器有机会调度其他的已就绪的协程。但是如果一个Goroutine如果一直不阻塞于上面的条件，例如就是简单的无限循环，就会饿死其他已就绪的协程，很显然这是不公平的。因此，Go调度器实现了一套协作式抢占的机制。每个协程在创建时，都继承了时间片，当时间片到期后，会被调度器置位。当这个Goroutine进行函数调用，执行检查是否需要增长栈空间时，就会进入runtime的代码，执行调度器，从而达到了抢占的目的。

```c
G->stackLimit = 0xfffffffffffffade
```

```c
# function Prologue
foo:
mov %fs:-8, %RCX // load G descriptor from TLS
cmp 16(%RCX), %RSP // compare the stack limit and RSP
jbe morestack // jump to slow-path if not enough stack
...
```

#### 公平小结

![img](/blog/img/goroutine_scheduler/fairness_hierarchy.png)

### 无尽栈(infinite stack)

#### 为什么需要无尽栈？

- 以最简单的方式支持各种弹性需求

- 对于海量的协程来说，地址空间并不是无限的（尤其是32位的系统）。

- 固定栈空间，小了不够用，大了浪费，没办法支持可伸缩(scalability)。

#### 分离栈还是连续栈？

- 分离栈

  - 初始分配4k, 如果协程栈不够用，另外分配一块内存，以链表的方式追加到原始栈内存之后。

  - 使用P管理stack cache, 减少栈内存分配的开销。

- 连续栈

  - 初始分配2k，如果协程栈不够用，分配大小是原始栈内存两倍大的内存，copy原始栈内存里的内容，到新分配的内存中。

  - 使用P管理stack cache, 减少栈内存分配的开销。

- 共享栈（libco）？

  - 对于不是正在运行的协程，把它们的栈内存统统紧密的存储进share stack，在协程被调度要运行时，从share stack中取出该协程的栈内容，还原该协程的调用栈，典型的以时间换空间的做法。

![img](/blog/img/goroutine_scheduler/stack_cache.png)

#### 何时分配

协程栈主要是用来存储临时变量的，由于临时变量的具有块生命周期，随着函数调用压栈退栈，因此go编译器会在每个go函数调用的开头，插入一段检查栈内存是否够用的指令。

```assembly
// Function Prologue
void foo()
{
 if (RSP < TLS_G->stack_limit)
  morestack();
}
# Goroutine stacks
foo:
mov %fs:-8, %RCX         // load G descriptor from TLS
cmp 16(%RCX), %RSP     // compare the stack limit and RSP
jbe morestack             // jump to slow-path if not enough stack
sub $64, %RSP
...
mov %RAX, 16(%RSP)
...
add $64, %RSP
retq
...
morestack:                     // call runtime to allocate more stack
callq 
```

### 总结： goroutine的调度策略

![img](/blog/img/goroutine_scheduler/goroutine_scheduler.png)

​     golang以非常简单的语句，提供了非常强大的协程功能，而且在用户空间(runtime)中实现了一套非常复杂的协程调度系统，包括引入P的结构避免thread数量的的膨胀，同时拥有良好的局部性，引入Local Queue实现lock free，在遇到阻塞操作(channel, mutex, time, network IO)时，让出CPU，使用单独的线程去等待系统调用和c call函数的返回。实现无尽栈，权衡最小的开销满足以形形色色的需求。实现work steal，充分利用多核的能力，实现协程与P，P和CPU核心的亲和性，来减少缓存失效的开销提高执行效率。以sysmon后台线程周期性的检查网络IO，以一定的概率检查全局队列，实现协作式抢占等，尽可能的实现调度公平和效率之间的最佳平衡。

​    总之，golang通过组合使用一系列经典的理论和技术，实现了一套**可伸缩性(scability)**， **简洁(minimal API)**， **调度公平**， 支持**无尽栈空间(infinite stack)**的协程调度系统，大大简化了开发人员并发编程时，所要面对复杂度。

> You are what you say.

## goroutine调度对coding的影响

1. 死循环

```go
package main
import (
    "fmt"
    "runtime"
)
var GLOBAL_COUNT int
func foo() {
    GLOBAL_COUNT++
}
func probe() {
    for {
        fmt.Println("probe running....................")
    }
}
func main() {
    CPUNum := runtime.GOMAXPROCS(-1)
    fmt.Println(CPUNum)
    go probe()
    for i := 0; i < CPUNum; i++ {
        go foo()
    }
    fmt.Println(GLOBAL_COUNT)
    select {}
}
```

2. 内存耗尽

```go
package main
import "fmt"
type S struct {
    a, b int
}
// String implements the fmt.Stringer interface
func (s *S) String() string {
    return fmt.Sprintf("%s", s) // Sprintf will call s.String()
}
func main() {
    s := &S{a: 1, b: 2}
    fmt.Println(s)
}
```

3. channel
   - 长度不为0的channel，写协程先调
   - 长度为0的channel，读协程先调度

```go
    c1 := make(chan int)
    c2 := make(chan int, 1)
    var a string
    go func() {
        a = "xx"
        <-c1
        <-c2
        fmt.Println(a)
        <-c1
    }()
    c1 <- 1
    fmt.Println(a)
    a = "yy"
    c2 <- 1
    c1 <- 1
```

4. 每当创建一个goroutine时，先明确它何时退出。

```go
ch := somefunction()
go func() {
    for range ch { }
}()
```

5. 判断应用是否需要并发？ 

   - add 

   - 冒泡排序/归并排序 

   - find 

     代码和解释](https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html)

6. **显式同步，不要猜测goroutine的调度顺序。**

7. [不要以共享内存的方式去通信，而要以通信的方式去共享内存](https://golang.org/doc/effective_go.html#concurrency)

### 查看goroutine调度监控

`> $GODEBUG=schedtrace=1000 godoc -http=:6060  <cmd>  `

```shell
SCHED 36185ms: gomaxprocs=12 idleprocs=0 threads=14 spinningthreads=0 idlethreads=1 runqueue=1 [0 0 0 0 0 0 0 0 0 0 0 0]
```

# Questions:

- Goroutine 的栈空间目前是二倍增长的方式进行扩张，在函数开始时进行检查是否需要扩张的(编译器插入的代码)，那么什么时候进行收缩呢？

- Golang准备在1.14对调度机制进行调整，调整成非协作式抢占，抢占如何实现？会遇到哪些挑战？

- GC期间，goroutine如何调度？

# 相关资料

- [Go scheduler: Implementing language with lightweight concurrency](https://assets.ctfassets.net/oxjq45e8ilak/48lwQdnyDJr2O64KUsUB5V/5d8343da0119045c4b26eb65a83e786f/100545_516729073_DMITRII_VIUKOV_Go_scheduler_Implementing_language_with_lightweight_concurrency.pdf) [[视频](https://www.youtube.com/watch?v=-K11rY57K7k)]

- [Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit)

- [Why goroutines are not lightweight threads?](https://codeburst.io/why-goroutines-are-not-lightweight-threads-7c460c1f155f)

- [Go语言调度器和goroutine](https://draveness.me/golang/concurrency/golang-goroutine.html)

- [Scheduling In Go : Part I - OS Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)

- [Scheduling In Go : Part II - Go Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)

- [Scheduling In Go : Part III - Concurrency](https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html)

- [Proposal: Non-cooperative goroutine preemption](https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md)

- [Go's work-stealing scheduler](https://rakyll.org/scheduler/)

- [The Go scheduler](http://morsmachine.dk/go-scheduler)

- [The Go netpoller](https://morsmachine.dk/netpoller)

- [Go: How Does the Goroutine Stack Size Evolve?](https://medium.com/a-journey-with-go/go-how-does-the-goroutine-stack-size-evolve-447fc02085e5)

- [也谈goroutine调度器](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)

- [深入解析Go](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.0.html)

- [Go并发机制](https://github.com/k2huang/blogpost/blob/master/golang/并发编程/并发机制/Go并发机制.md)

- [Debugging performance issues in Go programs](https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs)

- [CPU Affinity in Go](http://pythonwise.blogspot.com/2019/03/cpu-affinity-in-go.html)

- [Never start a goroutine without knowing how it will stop](https://dave.cheney.net/2016/12/22/never-start-a-goroutine-without-knowing-how-it-will-stop)

- [System Calls Make the World Go Round](https://manybutfinite.com/post/system-calls/)

- [Linux Syscall Reference](https://syscalls.kernelgrok.com/)

- [腾讯C++协程库：Libco](https://github.com/Tencent/libco)

- [《深入理解计算机系统》2th，第三章：程序的机器级表示](https://piazza.com/class_profile/get_resource/j7ly9riuca97on/ja86xbbpp0b73b)