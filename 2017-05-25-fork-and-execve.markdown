---
layout:     post
title:      "fork函数 与 execve函数"
subtitle:   "尽量详细的，深入的辨析一下 fork函数 和 execve函数。"
date:       2017-05-25
author:     "ChenWenKe"
header-img: "img/post_csapp/vm.jpg"
tags:
    - Linux
    - OS
---

## fork 函数 和 execve函数
fork 函数是产生新进程时要调用的函数（系统调用，准确的说是稍微封装的系统调用）； execve是在进程要加载执行新程序时，调用的系统调用。 这里请注意一下进程与程序的区别。 

#### fork出的子进程与父进程的异同

Besides the open files, numerous other properties of the parent are inherited by the
child:
- Real user ID, real group ID, effective user ID, and effective group ID
- Supplementary group IDs
- Process group ID
- Session ID
- Controlling terminal
- The set-user-ID and set-group-ID flags
- Current working directory
- Root directory
- File mode creation mask
- Signal mask and dispositions
- The close-on-exec flag for any open file descriptors
- Environment
- Attached shared memory segments
- Memory mappings
- Resource limits
The differences between the parent and child are
- The return values from fork are different.
- The process IDs are different.
- The two processes have different parent process IDs: the parent process ID of the
child is the parent; the parent process ID of the parent doesn’t change.
- The child’s tms_utime, tms_stime, tms_cutime, and tms_cstime values
are set to 0 (these times are discussed in Section 8.17).
- File locks set by the parent are not inherited by the child.
- Pending alarms are cleared for the child.
- The set of pending signals for the child is set to the empty set.

#### 使 fork 失败的两个主要原因

1. if too many processes are already in
the system, which usually means that something else is wrong.

2. if the total number of processes for this real user ID exceeds the system’s limit. CHILD_MAX specifies the maximum number of simultaneous processes per real user ID.

#### fork 的两种用法

1. When a process wants to duplicate itself so that the parent and the child can
each execute different sections of code at the same time. This is common for
network servers—the parent waits for a service request from a client. When the
request arrives, the parent calls fork and lets the child handle the request. The
parent goes back to waiting for the next service request to arrive.

2. When a process wants to execute a different program. This is common for
shells. In this case, the child does an exec (which we describe in Section 8.10)
right after it returns from the fork.

#### execve函数

- Process ID and parent process ID
- Real user ID and real group ID
- Supplementary group IDs
- Process group ID
- Session ID
- Controlling terminal
- Time left until alarm clock
- Current working directory
- Root directory
- File mode creation mask
- File locks
- Process signal mask
- Pending signals
- Resource limits
- Nice value
- Values for tms_utime, tms_stime, tms_cutime, and tms_cstime

The handling of open files depends on the value of the close-on-exec flag for each
descriptor. Every open descriptor in a process has a close-on-exec flag. If this flag
is set, the descriptor is closed across an exec. Otherwise, the descriptor is left open
across the exec. The default is to leave the descriptor open across the exec unless we
specifically set the close-on-exec flag using fcntl.


POSIX.1 specifically requires that open directory streams be closed across an exec. This is normally done by the opendir function calling fcntl to set the close-on-exec flag for the descriptor corresponding to the open directory stream.

**Note that the real user ID and the real group ID remain the same across the exec,
but the effective IDs can change**, depending on the status of the set-user-ID and the set-
group-ID bits for the program file that is executed. If the set-user-ID bit is set for the
new program, the effective user ID becomes the owner ID of the program file.
Otherwise, the effective user ID is not changed (it’s not set to the real user ID). The
group ID is handled in the same way.

#### fork 函数，execve函数与虚拟存储器

> 当 fork 函数被当前进程调用时， 内核为新进程创建各种数据结构，并分配给它一个唯一的 PID. 为了给这个新进程创建虚拟存储器，它创建了当前进程的 mm_struct, 区域结构和页表的原样拷贝。它将两个进程中的每个页面都标记为只读，并将两个进程中每个区域结构都标记为私有的写时拷贝。 

这就是所，当子进程产生时，它并没有把父进程在当前存储器中的代码和数据拷贝一份（并分配另外的存储器空间），而是创建了控制子进程的PID以及一份存储器映射,然后写时拷贝（copy-on-write）. 

> 子进程通过 execve 系统调用启动加载器。 加载器删除子进程现有的存储器段，并创建一组新的代码，数据，堆和栈段。 新的栈段和堆段被初始化为 零。 通过将虚拟存储器地址空间中的页映射到可执行文件的也大小的偏（chunk），新的代码和数据段被初始化为可执行文件的内容。除了一些头部信息，在加载过程中没有任何从磁盘到存储器的数据拷贝。 

#### 参考资料
- 《Unix环境高级编程 3th》
- 《深入理解计算机系统 原书2th》