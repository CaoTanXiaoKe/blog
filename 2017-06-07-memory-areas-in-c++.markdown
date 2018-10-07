---
layout:     post
title:      "「译」C++中的内存区"
subtitle:   "对C++中的内存区，栈区, 堆区, 自由存储区，只读数据区, 全局静态区进行较详细的辨析。"
date:       2017-06-07
author:     "ChenWenKe"
header-img: "img/post-bg-zhuan-liwei.jpg"
tags:
    - 译文
    - C++
    - OS
---

## C++ 中的内存区

---

> Const Data: The const data area stores string literals and other data whose values are known at compile time.  No objects of class type can exist in this area.  All data in this area is available during the entire lifetime of the program. Further, all of this data is read-only, and the results of trying to modify it are undefined. This is in part because even the underlying storage format is subject to arbitrary optimization by the implementation.  For example, a particular compiler may store string literals in overlapping objects if it wants to.

- 只读数据区: 
只读数据区存储一些字符串常量和一些其它的在编译时就能确定的值，这个区域不存在对象（由类实例化出的对象）。这个区域的数据的生存期是整个程序。更进一步说，这个区域里面所有数据都是只读的，任何试图改变这个区域里面数据的行为都是未定义的（虽然的确能够更改）。这主要是因为有些规则由于实现时的优化可能没被遵守。例如一个特定的编译器可能以层叠的方式存储字符串。（也就是说程序不同区域中定义的相同的字符串可能在内存中只有一份）。

--- 

> Stack: The stack stores automatic variables. Typically allocation is much faster than for dynamic storage (heap or free store) because a memory allocation involves only pointer increment rather than more complex management.  Objects are constructed immediately after memory is allocated and destroyed immediately before memory is deallocated, so there is no opportunity for programmers to directly manipulate allocated but uninitialized stack space (barring willful tampering using explicit dtors and placement new).

- 栈区:      
栈区存储自动变量，特别是栈区分配内存的速度比动态内存分配快的多（不管是在堆区还是在自由存储区）。这是因为栈区分配内存只需要指针增加一下就好了，而动态内存分配需要复杂的内存分配管理（查找空闲链表，做标记，划分，合并等）。在栈区内，为对象分配内存后，会立即调用该对象的构造函数进行初始化，对象销毁后立即释放该对象所占用的内存。所以根本没有给程序员直接管理栈区内存（除了未初始化的）的机会。当就像你知道的规则天生就有例外，用 explicit dtors(显式析构函数)和 placement new(定位 new)可以强势突破这个规则。 

---

> Free Store: The free store is one of the two dynamic memory areas, allocated/freed by new/delete.  Object lifetime can be less than the time the storage is allocated; that is, free store objects can have memory allocated without being immediately initialized, and can be destroyed without the memory being immediately deallocated.  During the period when the storage is allocated but outside the object's lifetime, the storage may be accessed and manipulated through a void* but none of the proto-object's nonstatic members or member functions may be accessed, have their addresses taken, or be otherwise manipulated.

- 自由存储区: 
自由存储区是两种动态内存分配区域中的一种，正如聪明的你所能想到的，另一种是 heap 啦。自由存储区的内存由 new/delete 进行申请和释放。对象的生命周期可以比该对象持有的内存的生命期更短一些，也就是说对象生命周期结束后并不立即释放内存。另一种说法是：在对象的内存分配后，可以不立即进行初始化，在对象销毁以后，可以不立即释放内存。在对象的生命周期已经结束了，但是内存还没释放的这段时间里，可以通过 void* 指针操作这块内存，，但是不允许对对象的 no-static成员进行访问，取地址以及其它操作。

---

> Heap: The heap is the other dynamic memory area, allocated/freed by malloc/free and their variants.  Note that while the default global new and delete might be implemented in terms of malloc and free by a particular compiler, the heap is not the same as free store and memory allocated in one area cannot be safely deallocated in the other. Memory allocated from the heap can be used for objects of class type by placement-new construction and explicit destruction.  If so used, the notes about free store object lifetime apply similarly here.

- 堆区: 
正如你所知道的那样，堆区就是动态分配内存区域中的“另一种”，由 malloc/free 以及它们的变形形式进行申请和释放。注意有些编译器会使用 malloc/free来实现全局默认的 new/delete，但是自由存储区和堆区并不一样。因此 用malloc申请的内存，用delete进行释放是不安全的（事实上是及其危险的），反之亦然。但是在构造对象时也可以使用堆区的内存，哈哈，没错就是用 explicit destruction (显式析构函数) 和 placement-new(定位new)。如果这样做，自由存储区中关于对象生命周期中的规则在这儿也同样适用。 

---

> Global/Static: Global or static variables and objects have their storage allocated at program startup, but may not be initialized until after the program has begun executing.  For instance, a static variable in a function is initialized only the first time program execution passes through its definition.  The order of initialization of global variables across translation units is not defined, and special care is needed to manage dependencies between global objects (including class statics).  As always, uninitialized proto-objects' storage may be accessed and manipulated through a void* but no nonstatic members or member functions may be used or referenced outside the object's actual lifetime.

- 全局静态区: 
全局或静态变量和对象在程序启动时就会分配内存空间，但是可能直到程序开始执行后才会被初始化。例如，一个在函数中的中的static变量（也就是 local static变量）直到在程序第一次执行到它的定义的时候才会初始化它。Effective C++中是这样描述的：C++保证，函数内的local static对象会在“该函数被调用期间”“首次遇上该对象的定义式”时被初始化。C++ 对定义于不同编译单元内的 no-loacl static 对象（及所有的全局变量）的初始化次序并无明确定义。特别要注意的是 全局对象（包括静态对象）之间的依赖关系是需要特殊管理的，例如可以像 Effective C++中所提供的方案，把它们分别写到一个函数里，此时就变成了 local static ，并返回指向它们的引用。所以，这也说明了全局静态区内的对象跟 free store里对对象生命周期的规定一样，未初始化的及超出对象生命周期的这块内存可以用 void*指针进行操作，但是对象的非静态成员变量和非静态成员函数不能被访问，取地址，引用及其它操作。

---

#### 附加一个小 Demo

```cpp
//main.cpp
int a = 0; 	// 全局静态区 
char *p1; 	// 全局静态区
main()
{
int b; 	// 栈
char s[] = "abc"; // s在栈,"abc"在常量区
char *p2; // 栈
char *p3 = "123456"; // 123456\0在常量区，p3在栈。
static int c = 0； // 全局静态区
p1 = (char *)malloc(10);
p2 = (char *)malloc(20); 	// 分配得来得10和20字节的区域就在堆区。
strcpy(p1, "123456");  // 123456\0放在常量区，编译器可能会将它与p3所指向的"123456"
优化成一个地方。
}

```

<br/>
[原文地址](http://www.gotw.ca/gotw/009.htm)
<br/>