---
layout:     post
title:      "漫谈：虚拟存储器"
subtitle:   "虚拟存储器的作用，基本原理，Intel Core i7/Linux存储器系统简述，mmap共享内存区小例"
date:       2017-04-20
author:     "ChenWenKe"
header-img: "img/post_csapp/vm.jpg"
tags:
    - Linux
    - OS
---

## 什么是虚拟存储器？(What)

为了更加有效地管理存储器并且少出错，现代系统提供了一种对主存的抽象概念， 叫做**虚拟存储器(VM)**。虚拟存储器是硬件异常，硬件地址翻译， 主存，磁盘文件和内核软件的完美交互， 它为每个进程提供了一个大的， 一致的私有地址空间。 通过一个很清晰的机制，虚拟存储器提供了三个重要的能力：

1. 它将主存看成是一个存储在磁盘上的地址空间的高速缓存，在主存中只保留活动区域，并且根据需要在磁盘和主存之间来回传送数据，通过这种方式它高效的利用了主存。 
2. 它为每个进程提供了一致的地址空间，从而简化了存储器管理。 
3. 它保护了每个进程的地址空间不被其他进程破坏。 

**总结：虚拟存储器就是对主存的抽象。**

## 虚拟存储器的作用？(Why)

很显然, 虚拟存储器提供的**三个重要的能力** 就是虚拟存储器的主要作用。 另外，

- 简化链接：独立的地址空间允许每个进程的存储器映像使用相同的基本格式，而不管代码和数据实际存放在物理存储器的何处。
- 简化加载： 加载器从不实际拷贝任何数据从磁盘到存储器。而是在页面初次被引用时，虚拟存储器系统会按照需要自动地调入数据页。 
- 简化共享：独立地址空间为操作系统提供了一个管理用户进程和操作系统自身之间共享的一致机制（如：操作系统内核代码，程序之间的共享库）。 
- 简化存储器分配：程序申请动态内存分配时，操作系统为它分配连续的 k 个连续的虚拟存储器页面，并且将它们映射到物理存储器中任一位置的 k 个任一的物理页面。 


## 虚拟存储器的基本原理 (How)

#### CPU通过生成虚拟地址进行访问主存。 
![use virtual addressing](/blog/img/post_csapp/useVirsualAddress.PNG)

在一个带虚拟存储器的系统中，CPU 从一个有 N=2^n 个地址的地址空间中生成虚拟地址，这个地址空间称为**虚拟地址空间（virtual address space）**. 

地址空间的概念是很重要的，因为它清楚地区分了数据对象（字节）和他们的属性（地址）。一旦认识到了这种区别，那么我们就可以将其推广， 允许每个数据对象有多个独立的地址，其中每个地址都选自一个不同的地址空间。这就是虚拟存储器的基本思想。主存中的每个字节都有一个选自虚拟地址空间的虚拟地址和一个选自物理地址空间的物理地址。

就概念上来说，虚拟存储器（VM）是磁盘上的N个连续的字节大小的单元组成的数组。每个字节有一个虚拟地址，这个虚拟地址就作为相应数组内容的索引。主存可以看做是磁盘的高速缓存。 磁盘上的数据被分割成块，这些块作为磁盘和主存之间的传输单元。VM系统把虚拟存储器分割为称为**虚拟页（Virtual Page,VP）**的大小固定的块。物理存储器被分割为和虚拟页等大小的**物理页（Physical Page, PP）**. 

![mainMemoryAsCache](/blog/img/post_csapp/mainMemoryAsCache.PNG)

#### 虚拟页面集合有三种状态： 
1. 未分配的，没有任何数据和它们相关联，当然不占用任何磁盘空间。 
2. 缓存的：当前缓存在物理存储器中的已分配页。 
3. 未缓存的：没有缓存在物理存储器中的已分配页。

#### 虚拟存储器通过页表把虚拟地址和物理地址联系起来。
![PageTable](/blog/img/post_csapp/pageTable.PNG)

现代操作系统都使用: 只有页面不命中时，才进行页面的换入换出的**按需页面调度(demand paging)**方式.

#### 缺页：
![paging](/blog/img/post_csapp/paging1.PNG)

#### 页面调度：
![paging](/blog/img/post_csapp/paging2.PNG)

### 再看虚拟存储器的作用
#### 虚拟存储器做为存储器管理的工具
实际上，操作系统为每个进程提供了一个独立的页表， 因而也就是一个独立的虚拟地址空间。 多个虚拟页面可以映射到同一个共享物理页面上。 按需调度和独立的虚拟地址空间的结合，对系统中存储器的使用和管理造成了深远的影响。 特别地， VM 简化了链接和加载， 代码和数据共享， 以及应用程序的存储器分配。 

![The operating maintains a separatePageTable](/blog/img/post_csapp/separatePageTable.PNG)

- **简化链接**：独立的地址空间允许每个进程的存储器映像使用相同的基本格式，而不管代码和数据实际存放在物理存储器的何处。
- **简化加载**： 加载器从不实际拷贝任何数据从磁盘到存储器。而是在页面初次被引用时，虚拟存储器系统会按照需要自动地调入数据页。 
- **简化共享**：独立地址空间为操作系统提供了一个管理用户进程和操作系统自身之间共享的一致机制（如：操作系统内核代码，程序之间的共享库）。 
- **简化存储器分配**：程序申请动态内存分配时，操作系统为它分配连续的 k 个连续的虚拟存储器页面，并且将它们映射到物理存储器中任一位置的 k 个任一的物理页面。 

#### 虚拟存储器做为存储器保护的工具
因为每次CPU生成一个地址时，地址翻译硬件都会都一个PTE（Page Table Enty, 页表条目），所以通过在 PTE 上添加一些额外的许可位来控制对虚拟页面内容的访问。 

![memory protection](/blog/img/post_csapp/memoryProtection.PNG)

#### TLB 和 多级页表
为了降低访问页表的开销，在 MMU 中包含一个关于 PTE 的小的缓存。 称为**翻译后备缓冲器(Translation Lookaside Buffer, TLB)**. 
![TLB](/blog/img/post_csapp/TLB.PNG)

由于页表条目很多，现实情况中常用多级页表的层级结构对页表进行压缩。 
![two levels page tables](/blog/img/post_csapp/twoLevelsPageTable.PNG)

## Intel Core i7/Linux存储器系统
Core i7 实现支持 48 位（256 TB）虚拟地址空间和 52位 (4 PT) 物理地址空间，还有一个兼容模式，支持 32 位 (4GB)虚拟和物理地址空间。 页大小在启动时被配置为 4KB 或 4MB. Linux 使用的是 4KB的页。 另外，Core i7 采用四级页表层次结构。 

#### Linux 虚拟存储器系统
Linux 为每个进程维护了一个单独的虚拟地址空间。
![Linux Virtual Memory](/blog/img/post_csapp/LinuxVirtualMemory.PNG)

#### Linux 虚拟存储器区域
Linux 将虚拟存储器组织成一些区域（也叫做段）的集合。 一个区域(area)就是已经存在着的（已分配的）虚拟存储器的连续片(chunk)， 这些页是以某种方式相关联的。 例如， 代码段，数据段， 堆， 共享库段， 以及用户栈都是不同的区域。 每个存在的虚拟页面都保存在某个区域中，而不属于某个区域的虚拟页是不存在的， 并且不能被进程引用。 区域的概念很重要， 因为它允许虚拟地址空间有间隙。 内核不用记录那些不存在的虚拟页，而这样的页也不占用存储器，磁盘或者内核本身的任何额外资源。 

#### 进程中虚拟存储器区域的内核数据结构
![进程中虚拟存储器区域的内核数据结构](/blog/img/post_csapp/LinuxOrganizesVM.PNG)

- vm_start: 指向这个区域的起始处。
- vm_end: 指向这个区域的结束处。
- vm_prot: 描述这个区域内包含的所有页的读写许可权限。
- vm_flags: 描述这个区域内包含的页面是与其他进程共享的，还是这个进行私有的(还描述了其他一些信息)。
- vm_next: 指向链表中下一个区域结构。 


#### Linux 缺页处理：
假如MMU(存储器管理单元)在试图翻译某个虚拟地址A时，触发了一个缺页。 这个异常导致控制转移到内核的缺页处理程序，处理程序随后搜索区域结构链表，进行缺页处理。
![Linux Paging](/blog/img/post_csapp/LinuxPaging.PNG)

#### 存储器映射
Linux(以及其他一些形式的 Unix)通过将一个虚拟存储器区域与一个磁盘上的对象（object）关联起来，以初始化这个虚拟存储器区域的内容，这个过程称为**存储器映射(memory mapping)**。 虚拟存储器区域可以映射到两种类型的对象中的一种。 
- Unix 文件系统中的普通文件： 一个区域可以映射到一个普通文件的连续部分， 例如一个可执行目标文件。文件区（section）被分成页大小的片，每个片包含一个虚拟页面的初始内容。因为是按需进行调度，所以这些虚拟页面没有实际交换进入物理存储器，直到 CPU 第一次引用到页面（即发射一个虚拟地址，落在地址空间这个页面范围之内）。 如果区域比文件区要大，那么就用零来填充这个区域的余下部分。 

- 匿名文件： 一个区域也可以映射到一个匿名文件， 匿名文件是由内核创建的，包含的是二进制零。 

无论在哪种情况下，一旦一个页面被初始化了，它就在一个由内核维护的专门的交换文件（swap file）之间换来换去。 交换文件也叫做交换空间（swap space）或者交换区域（swap area）. 需要意识到的很重要的一点是， 在任何时刻， 交换空间都限制着当前运行着的进程能够分配的虚拟页面总数。
![Memory Maping](/blog/img/post_csapp/memoryMaping.PNG)


## 共享存储器

共享对象：
![A shared object](/blog/img/post_csapp/aSharedObject.PNG)

私有对象：写时拷贝
![A private object](/blog/img/post_csapp/aPrivateObject.PNG)

## 动态内存分配简述
![the heap](/blog/img/post_csapp/theHeap.PNG)
![heap List](/blog/img/post_csapp/heapList.PNG)

---

## 简述一个程序的执行(Use)

```cpp
#include <stdio.h>
#include <unistd.h>
#include <semaphore.h>  // Posix semaphores
#include <sys/mman.h>   // Posix shared memory
#include <stdlib.h>
#include <sys/types.h>  // for open()
#include <sys/stat.h>
#include <fcntl.h>

int count = 0; 
int main(int argc, char **argv)
{
    int nloop = 5;
    int i;  
    setbuf(stdout, NULL);   // stdout is unbuffered
    if (fork() == 0) {  // child
        for (i = 0; i < nloop; i++) {
            printf("child: %d\n", count++);
            sleep(1); 
        }
        exit(0);
    }
    // parent 
    for (i = 0; i < nloop; i++) {
        printf("parent: %d\n", count++);
        sleep(1);
    }
    exit(0); 
}
```
#### 编译链接
`预处理 --> 编译 --> 汇编 --> 链接 = 可执行的目标文件`
![ELF](/blog/img/post_csapp/ELF.PNG)

#### 加载执行
`fork一个新进程 --> execve 加载可执行目标文件（动态链接） --> 被访问 --> 缺页 --> 页调度 --> 执行`

![run-time Memory image](/blog/img/post_csapp/run-timeMemory.PNG)

--- 

- 在链接结束时，程序中的每个指令和全局变量都有唯一的运行时地址了。 

- Unix 系统中的每个进程都运行在一个进程的上下文中，有自己的虚拟地址空间。当外壳运行一个程序时，父外壳进程生成一个子进程，它是父进程的一个复制品。 子进程通过 execve 系统调用启动加载器。 加载器删除子进程现有的虚拟存储器段，并创建一组新的代码，数据，堆和栈段。新的栈和堆段被初始化为零（请求二进制零的）。 通过将虚拟地址空间中的页映射到可执行文件页大小的片（chunk）, 新的代码和数据段被初始化为可执行的内容。 最后，加载器跳转到_start地址，它最终会调用应用程序的 main函数。除了一些头部信息，在加载过程中没有任何从磁盘到存储器的数据拷贝。直到 CPU 引用一个被映射的虚拟页才会进行拷贝，此时，操作系统利用它的页面调度机制自动将页面从磁盘传送到存储器。  


#### 进程间通信：共享存储空间

* 没有共享时的父子进程：

```cpp
#include <stdio.h>
#include <unistd.h>
#include <semaphore.h>  // Posix semaphores
#include <sys/mman.h>   // Posix shared memory
#include <stdlib.h>
#include <sys/types.h>  // for open()
#include <sys/stat.h>
#include <fcntl.h>

int count = 0; 
int main(int argc, char **argv)
{
    int nloop = 5;
    int i;  
    setbuf(stdout, NULL);   // stdout is unbuffered
    if (fork() == 0) {  // child
        for (i = 0; i < nloop; i++) {
            printf("child: %d\n", count++);
            sleep(1); 
        }
        exit(0);
    }
    // parent 
    for (i = 0; i < nloop; i++) {
        printf("parent: %d\n", count++);
        sleep(1);
    }
    exit(0); 
}
```

输出：

```cpp
parent: 0
child: 0
parent: 1
child: 1
parent: 2
child: 2
parent: 3
child: 3
parent: 4
child: 4
```

<br/>
* 用mmap映射共享存储区：

```cpp
#include <stdio.h>
#include <unistd.h>
#include <semaphore.h>  // Posix semaphores
#include <sys/mman.h>   // Posix shared memory
#include <stdlib.h>
#include <sys/types.h>  // for open()
#include <sys/stat.h>
#include <fcntl.h>

#define FILE_MODE (S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH)

struct shared {
    sem_t   mutex;      // the mutex: a posix memory-based semaphore
    int     count;      // and the counter
} shared; 

int main(int argc, char **argv)
{
    int fd, i, nloop = 5; 
    struct shared   *ptr;  

    ptr = (struct shared*) mmap(NULL, sizeof(struct shared), PROT_READ|PROT_WRITE, 
    		MAP_SHARED|MAP_ANON, 0, 0); 
    close(fd); 
    // initialize semaphore that is shared between processes
    sem_init(&ptr->mutex, 1, 1); 

    setbuf(stdout, NULL);   // stdout is unbuffered
    if (fork() == 0) {  // child
        for (i = 0; i < nloop; i++) {
            sem_wait(&ptr->mutex); 
            printf("child: %d\n", ptr->count++);
            sem_post(&ptr->mutex);
            sleep(1); 
        }
        exit(0);
    }
    // parent 
    for (i = 0; i < nloop; i++) {
        sem_wait(&ptr->mutex); 
        printf("parent: %d\n", ptr->count++);
        sem_post(&ptr->mutex); 
        sleep(1);
    }
    exit(0); 
}
```

输出：

```cpp
parent: 0
child: 1
parent: 2
child: 3
parent: 4
child: 5
parent: 6
child: 7
parent: 8
child: 9
```

#### Linux 系统中的物理内存和虚拟内存

物理内存自然不用多说，就是实际的RAM大小。 可以通过 `free -m` 查看物理内存是使用和空闲状况。如：

```
             total       used       free     shared    buffers     cached
Mem:          993M       898M        95M       628K       154M       446M
-/+ buffers/cache:       297M       696M
Swap:           0B         0B         0B
```
系统中的物理内存总大小为：993M, 已经使用898M（包括缓冲区和缓存）, 剩余95M. 如果除去缓冲区和缓存，已经使用的物理内存实际是297M, 剩余696M. Swap 文件的大小为 0. 


虚拟内存受限于Swap分区和物理内存的大小。 Swap分区里面存放的交换页是页面在内存中的直接映像（不同于一般的磁盘存储）。 使用`ulimite -a` 可以查看系统对内存的限制，分别加上-H, -S可以查看系统的软/硬限制。
```
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 7884
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 7884
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

--- 

<br/>
[图片来自csapp 2th 英文版]
## 参考资料

- 《深入理解计算机系统 2th(原书3th)》（CSAPP）
- 《计算机操作系统 4th》 汤小丹等著
- 《Unix网络编程卷二：进程间通信》（UNP）
- [Linux 内存机制详解宝典](http://www.jb51.net/LINUXjishu/34605.html)
<br/>
