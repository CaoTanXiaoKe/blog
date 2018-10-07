---
layout:     post
title:      "用互斥量和条件变量实现生产者消费者队列"
subtitle:   "互斥量，条件变量，生产者-消费者队列， 线程同步"
date:       2017-07-15
author:     "ChenWenKe"
header-img: "img/post-bg-zhuan-liwei.jpg"
tags:
    - C++
    - Linux
    - OS
    - 多线程
---

## 用互斥量和条件变量实现生产者-消费者队列
- 关于线程的其他同步方式，生产者-消费者队列的广泛应用的讨论，已经在[另一篇]("2017/07/05/linux-thread-producer-customer/")博客中提到了。 

### 代码
- 下面我在一个项目中，用C++封装的一个生产者消费者-队列。（使用互斥量和条件变量实现同步）

```cpp
// 头文件 LogQueue.h
#ifndef __LOGQUEUE_H__
#define __LOGQUEUE_H__

#include "data.h"		// 里面定义了 MLogRec 结构体
using namespace std;

class LogQueue
{
public:
   LogQueue();      // 初始化条件变量， 条件变量
   ~LogQueue();     // 反初始化条件变量， 条件变量
   LogQueue& operator<<(MLogRec const log);     // 把一条记录存入存储队列
   LogQueue& operator>>(MLogRec& log);          // 从存储队列取出一条记录

private:
   list<MLogRec> m_log;             // 要存储的数据
   pthread_mutex_t m_mtx;           // 互斥量
   pthread_cond_t m_notFull;        // 未满 - 条件变量
   pthread_cond_t m_notEmpty;       // 非空 - 条件变量

   enum { QUEUESIZE = 100}; 
};

#endif  // __LOGQUEUE_H__
```
<br/>
```cpp
// 实现文件： LogQueue.cpp
#include "LogQueue.h"

// 初始化互斥量，条件变量
LogQueue::LogQueue() :  m_mtx(PTHREAD_MUTEX_INITIALIZER), 
                        m_notFull(PTHREAD_COND_INITIALIZER),
                        m_notEmpty(PTHREAD_COND_INITIALIZER)
{

}


// 如果动态分配，反初始化互斥量，条件变量
LogQueue::~LogQueue()
{
   
}

// 重载 << 运算符： 把一条记录放入存储队列
LogQueue& LogQueue::operator<<(MLogRec const log)
{
   pthread_mutex_lock(&m_mtx); 
   while(m_log.size() >= QUEUESIZE)
   {
       pthread_cond_wait(&m_notFull, &m_mtx); 
   }
   m_log.push_back(log); 
   pthread_cond_broadcast(&m_notEmpty); 
   pthread_mutex_unlock(&m_mtx); 

   return *this; 
}


// 重载 >> 运算符：从存储队列中取出一条记录
LogQueue& LogQueue::operator>>(MLogRec& log)
{
    pthread_mutex_lock(&m_mtx); 
    while(m_log.size() == 0)
    {
        pthread_cond_wait(&m_notEmpty, &m_mtx); 
    }
    
    log = m_log.front(); 
    m_log.pop_front(); 

    pthread_cond_broadcast(&m_notFull); 
    pthread_mutex_unlock(&m_mtx); 

    return *this; 
} 
```

### 关于互斥量和条件变量的总结
- 互斥量从本质上说是一把锁，在访问共享资源前对互斥量进行设置（加锁），在访问完后释放（解锁）互斥量。 
- 条件变量给多个线程提供了一个会合的场所。条件变量与互斥量一起使用时，允许线程以无竞争的方式等待的特定的条件发生。条件变量本身是由互斥量保护的。线程在改变条件状态之前必须首先锁住互斥量。其他线程在获得互斥量之前不会察觉到这种改变，因为互斥量必须在锁定以后才能计算条件。 

### 信号量 VS 互斥量 VS 条件变量

1. 互斥锁必须总是由给它上锁的线程解锁，而信号量的 wait和post操作不必由同一个线程执行。

2. 互斥锁要么被锁住，要么被解开，和二值信号量类似。

3. 条件变量在发送信号时，如果没有线程等待在该条件变量上，那么信号将丢失；而信号量有计数值，每次信号量post操作都会被记录。

4. (信号量的)sem_post是各种同步技巧中，唯一一个能在信号处理程序中安全调用的函数。

5. 互斥锁是为上锁而优化的；条件变量是为等待而优化的；信号量既可用于上锁，也可用于等待，因此会有更多的开销和更高的复杂性 

<br/>
#### 参考资料
- 《Unix环境高级编程 3th》
- 《Unix网络编程 3th, 卷二》
- 《深入理解计算机系统 原书2th》

<br/>
<br/>