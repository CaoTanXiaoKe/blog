---
layout:     post
title:      "Core文件不能生成的几种原因"
subtitle:   "列举一下Core文件不能生成的几种原因，以及分析一下什么是Core文件, 什么是文件空洞。"
date:       2017-05-13
author:     "ChenWenKe"
header-img: "img/post_csapp/vm.jpg"
tags:
    - Linux
    - OS
---

#### core 文件不能生成的原因：

1. 进程是设置用户ID的，而且当前用户并非程序文件的所有者。 
2. 进程是设置组ID的，而且当前用户并非程序文件的组所有者。 
3. 用户没有写当前目录的权限。 
4. 文件已存在，而且用户对该文件没有写权限。 
5. 文件太大。即：要生成的core文件大小超过了 RLIMIT_CORE 的限制。 

前 4 种原因是因为权限问题，解决方案就是给予相应的权限。 第五种方法可以使用 `ulimit -c unlimited` 命令，使得系统对 core 文件的大小不做限制。ulimit 命令是内置于shell中的，所以这条命令对当前 shell 有效。 如果想让这条命令对当前用户有效，可以把这条命令写进 `.bashrc`配置文件，然后`source .bashrc`让系统重新读取一下 .bashrc 配置文件。 
<br/>

#### 什么是 core 文件

core文件的命名其实是个历史问题，一般在系统异常终止时, 系统会把该进程的内存映像（该文件名为 core）复制进当前工作目录的 core 文件中。 大多数的 Unix及Linux系统调试程序都使用 core文件检查程序终止时的状态。 例如：`gdb a.out core`。

```cpp
#include <stdio.h>

void func(int *ptr)
{
    *ptr = 0; 
}

int main()
{
    func(NULL); 

    return 0;
}

```
上面的程序试图对“NULL”进行赋值，会发生段错误，并转储。 

#### 文件空洞

普通文件中是可以包含空洞的， 空洞是由所设置的偏移量超过文件的末尾，并写入了某些数据后造成的。 

一个生成空洞的程序：

```c
/*************************************************************************
* File Name: fileHole.c
* Author: Chen WenKe
* Email: chenwenke666@gmail.com
* Blog: https://caotanxiaoke.github.io
* Created Time: Sat 13 May 2017 09:29:56 AM CST
*
* Description:
    创建一个具有空洞的文件：file.hole
 ************************************************************************/

#include <stdio.h>
#include <unistd.h>
#include <sys/stat.h>

const int FILE_MODE = (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);

char buf1[] = "abcdefghij";
char buf2[] = "ABCDEFGHIJ";

int main()
{
    int fd;

    if((fd = creat("file.hole", FILE_MODE)) < 0)
        printf("creat error");

    if(write(fd, buf1, 10) != 10)
        printf("buf write error"); // offset now = 10

    if(lseek(fd, 16384, SEEK_SET) == -1)
        printf("lseek error");  // offset now = 16384

    if(write(fd, buf2, 10) != 10)
        printf("buf2 write error"); // offset now = 16394

    return 0;
}
```

分别用 `ls -l file.hole` 和 `od -c file.hole` 查看一下：

```
$ ls -l file.hole
-rw-r--r-- 1 ubuntu ubuntu 16394 May 13 09:42 file.hole
$ od -c file.hole
0000000   a   b   c   d   e   f   g   h   i   j  \0  \0  \0  \0  \0  \0
0000020  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
*
0040000   A   B   C   D   E   F   G   H   I   J
0040012
$
```

**在带有空洞的文件被复制的时候，会把空洞用 0 填满。**

```
$ cat file.hole > file.nohole
$ ls -ls file.hole file.nohole
 8 -rw-r--r-- 1 ubuntu ubuntu 16394 May 13 09:42 file.hole
20 -rw-rw-r-- 1 ubuntu ubuntu 16394 May 13 09:51 file.nohole
```

上面两个文件虽然文件的长度(16394)相同, 但是有空洞的文件占用 8 个磁盘块，而无空洞的文件占用 20 磁盘块。

PS: 由于 core文件一般是进程出现错误时的存储器映像，所以有时也会有空洞存在于core文件中。 在这儿一并总结下。 

#### 参考资料
- 《Unix环境高级编程 3th》