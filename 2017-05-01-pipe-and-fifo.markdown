---
layout:     post
title:      "管道和FIFO(有名管道)"
subtitle:   "管道和FIFO的一些重要概念，以及实例代码。"
date:       2017-05-01
author:     "ChenWenKe"
header-img: "img/post-bg-think-deeply.jpg"
tags:
    - OS
---

## IPC(进程间通信)系列：管道和FIFO

##### 重要概念
FIFO又被称为是**有名管道**，管道和FIFO一般是半双工的，提供一个单路（单向）数据流。由两个文件描述符分别标记读和写。 

文件描述符可以通过父子进程传递，子进程自动继承父进程打开的文件描述符。 也可以通过**门**在无亲缘关系的进程间传递，甚至可以在客户端进程与服务器端进程间传递。 

管道直接连接着两端的进程，但是要注意**管道是由内核运作的**，即：管道建立在内核的地址空间中。 所以这就意味着，对管道每次的读写要陷入内核。

**我的机器环境：Ubuntu 14.04 双核64位**

##### 管道实例代码
父子进程之间，通过管道通信。 
- 客户端通过管道向服务器端发送文件名（绝对文件路径）
- 服务器端从管道读取客户端发送来的文件名
- 服务器端打开相应的文件，如果失败，通过管道向客户端发送错误消息；如果成功，发送相应的文件内容。 
- 客户端接收文件内容或错误消息，并输出到屏幕。 

```cpp
#include <unistd.h> // for pipe
#include <limits.h> // PIPE_BUF
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>  // for errno
#include <fcntl.h>  // for O_RDONLY

const int MAXLINE = 4096; 

void client(int, int), server(int, int); 

int main(int argc, char** argv)
{
    int pipe1[2], pipe2[2]; 
    pid_t childpid; 

    pipe(pipe1);    // create two pipes
    pipe(pipe2); 

    if((childpid = fork()) == 0)    // child  -- server 
    {
        close(pipe1[1]); 
        close(pipe2[0]); 

        server(pipe1[0], pipe2[1]); 
        exit(0); 
    }

    close(pipe1[0]); 
    close(pipe2[1]); 

    client(pipe2[0], pipe1[1]); 

    waitpid(childpid, NULL, 0);     // wait for child to terminate
    exit(0); 
}

void client(int readfd, int writefd)
{
    size_t  len; 
    ssize_t n; 
    char    buff[MAXLINE]; 

    fgets(buff, MAXLINE, stdin);    // read pathname 
    len = strlen(buff);     // fgets() guarantees null byte at end
    if(buff[len-1]='\n')
        len--;          // delete newline from fgets()

    write(writefd, buff, len);  // write pathname to IPC channel
    while((n = read(readfd, buff, MAXLINE)) > 0)
        write(STDOUT_FILENO, buff, n); 
}

void server(int readfd, int writefd)
{
    int     fd; 
    ssize_t n; 
    char    buff[MAXLINE]; 

    // read pathname from IPC channel
    if((n = read(readfd, buff, MAXLINE)) == 0)
        printf("end-of-file while reading pathname"); 
    buff[n] = '\0';     // null terminate pathname 
    
    if((fd = open(buff, O_RDONLY)) < 0)
    {
        // error: must tell client 
        snprintf(buff + n, sizeof(buff) - n, "can't open, %s\n", strerror(errno)); 
        n =  strlen(buff); 
        write(writefd, buff, n); 
    }
    else
    {
        // open succeeded: copy file to IPC channel
        while((n = read(fd, buff, MAXLINE)) > 0)
        {
            write(writefd, buff, n); 
            close(fd); 
        }
    }
}
```

<br/>

#### 管道实例代码

程序进程与shell命令产生的进程之间通信。 和一个例子实现的功能一样。不过，服务器端用 `cat`代替。
`popen`在调用进程和所指定的shell命令之间创建一个管道。 

```cpp
#include <unistd.h> // for pipe
#include <limits.h> // PIPE_BUF
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>  // for errno
#include <fcntl.h>  // for O_RDONLY

const int MAXLINE = 4096; 

int main(int argc, char**argv)
{
    size_t  n; 
    char    buff[MAXLINE];
    char    command[MAXLINE];   // shell command 
    FILE    *fp; 

    fgets(buff, MAXLINE, stdin);    // fgets() guarantees null byte at end 
    n = strlen(buff); 
    if(buff[n - 1] == '\n')
        n--;        // delete newline from fgets()
    snprintf(command, sizeof(command), "cat %s", buff); 
    fp = popen(command, "r"); 

    // copy from pipe to standard output
    while(fgets(buff, MAXLINE, fp) != NULL)
        fputs(buff, stdout); 

    pclose(fp); 
    exit(0); 
}
```
<br/>

#### FIFO实例代码

和上面的例子实现的功能一样，只是要指定两个路径名作为有名管道（FIFO）的名称。

父子进程间的有名管道。 

```cpp
#include <unistd.h> // for pipe
#include <limits.h> // PIPE_BUF
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>  // for errno
#include <fcntl.h>  // for O_RDONLY

const int MAXLINE = 4096; 
const char* FIFO1 = "/tmp/fifo.1"; 
const char* FIFO2 = "/tmp/fifo.2";
const mode_t FILE_MODE = (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH); 

void client(int, int); 
void server(int, int); 

int main(int argc, char** argv)
{
    int     readfd, writefd; 
    pid_t   childpid; 

    // create two FIFOs; OK is they already exist 
    if((mkfifo(FIFO1, FILE_MODE) < 0) && (errno != EEXIST))
        printf("can't create %s", FIFO1); 
    if((mkfifo(FIFO2, FILE_MODE) < 0) && (errno != EEXIST))
    {
        unlink(FIFO1);  // free FIFO1 
        printf("can't create %s", FIFO2); 
    }
    if((childpid = fork()) == 0)    // child - server
    {
        readfd = open(FIFO1, O_RDONLY, 0); 
        writefd = open(FIFO2, O_WRONLY, 0); 

        server(readfd, writefd); 
        exit(0);
    }

    // parent - client 
    writefd = open(FIFO1, O_WRONLY, 0); 	// the order to the two open() is important 
    readfd = open(FIFO2, O_RDONLY, 0); 

    client(readfd, writefd); 

    waitpid(childpid, NULL, 0);     // wait for child to terminate 

    close(readfd); 
    close(writefd); 

    unlink(FIFO1); 
    unlink(FIFO2); 
    exit(0); 
}



void client(int readfd, int writefd)
{
    size_t  len; 
    ssize_t n; 
    char    buff[MAXLINE]; 

    fgets(buff, MAXLINE, stdin);    // read pathname 
    len = strlen(buff);     // fgets() guarantees null byte at end
    if(buff[len-1]='\n')
        len--;          // delete newline from fgets()

    write(writefd, buff, len);  // write pathname to IPC channel
    while((n = read(readfd, buff, MAXLINE)) > 0)
        write(STDOUT_FILENO, buff, n); 
}

void server(int readfd, int writefd)
{
    int     fd; 
    ssize_t n; 
    char    buff[MAXLINE]; 

    // read pathname from IPC channel
    if((n = read(readfd, buff, MAXLINE)) == 0)
        printf("end-of-file while reading pathname"); 
    buff[n] = '\0';     // null terminate pathname 
    
    if((fd = open(buff, O_RDONLY)) < 0)
    {
        // error: must tell client 
        snprintf(buff + n, sizeof(buff) - n, "can't open, %s\n", strerror(errno)); 
        n =  strlen(buff); 
        write(writefd, buff, n); 
    }
    else
    {
        // open succeeded: copy file to IPC channel
        while((n = read(fd, buff, MAXLINE)) > 0)
        {
            write(writefd, buff, n); 
            close(fd); 
        }
    }
}

```
<br/>

和上面的例子功能一样，一台机器上没有亲缘关系的进程间通信。

```cpp
// filename: head.h
#include <unistd.h> // for pipe
#include <limits.h> // PIPE_BUF
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>  // for errno
#include <fcntl.h>  // for O_RDONLY

const int MAXLINE = 4096; 
const char* FIFO1 = "/tmp/fifo.1"; 
const char* FIFO2 = "/tmp/fifo.2";
const mode_t FILE_MODE = (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH); 
``` 

```cpp
// filename: server.c
#include "head.h"

void server(int, int); 

int main(int argc, char** argv)
{
    int readfd, writefd; 

    // create two FIFOs; OK if they already exist
    if ((mkfifo(FIFO1, FILE_MODE) < 0) && (errno != EEXIST))
        printf("can't create %s", FIFO1); 
    if((mkfifo(FIFO2, FILE_MODE) < 0) && (errno != EEXIST))
    {
        unlink(FIFO1);  // free FIFO1
        printf("can't create %s", FIFO2); 
    }
    readfd = open(FIFO1, O_RDONLY, 0); 
    writefd = open(FIFO2, O_WRONLY, 0); 

    server(readfd, writefd);
    exit(0); 
}

void server(int readfd, int writefd)
{
    int     fd; 
    ssize_t n; 
    char    buff[MAXLINE]; 

    // read pathname from IPC channel
    if((n = read(readfd, buff, MAXLINE)) == 0)
        printf("end-of-file while reading pathname"); 
    buff[n] = '\0';     // null terminate pathname 
    
    if((fd = open(buff, O_RDONLY)) < 0)
    {
        // error: must tell client 
        snprintf(buff + n, sizeof(buff) - n, "can't open, %s\n", strerror(errno)); 
        n =  strlen(buff); 
        write(writefd, buff, n); 
    }
    else
    {
        // open succeeded: copy file to IPC channel
        while((n = read(fd, buff, MAXLINE)) > 0)
        {
            write(writefd, buff, n); 
            close(fd); 
        }
    }
}
```

```cpp
// filename: client.c
#include "head.h"
void client(int readfd, int writefd); 

int main(int argc, char** argv)
{
    int readfd, writefd; 
    writefd = open(FIFO1, O_WRONLY, 0); 
    readfd = open(FIFO2, O_RDONLY, 0); 

    client(readfd, writefd); 

    close(readfd); 
    close(writefd); 

    unlink(FIFO1); 
    unlink(FIFO2); 
    exit(0); 
}

void client(int readfd, int writefd)
{
    size_t  len; 
    ssize_t n; 
    char    buff[MAXLINE]; 

    fgets(buff, MAXLINE, stdin);    // read pathname 
    len = strlen(buff);     // fgets() guarantees null byte at end
    if(buff[len-1]='\n')
        len--;          // delete newline from fgets()

    write(writefd, buff, len);  // write pathname to IPC channel
    while((n = read(readfd, buff, MAXLINE)) > 0)
        write(STDOUT_FILENO, buff, n); 
}
```

**注意：**必须先运行服务器端代码，建立FIFO通道，再运行客户端代码。 
<br/>

#### Shell 中建立管道和FIFO
在Shell中建立管道十分简单通过 `|` 即可。 例如统计 head.h的行数

> `$ cat head.h | wc -c`

在 Shell 中也可以通过 `mkfifo` 命令来建立 FIFO(有名管道)。例如：

> `$ mkfifo /tmp/ctxkfifo`
> `$ cat head.h > /tmp/ctxkfifo`

上面的命令不会返回（FIFO默认是阻塞的），直到从另一个Shell把数据读走。

> `$ wc -c /tmp/ctxkfifo` 

#### 概念性知识

内核为管道和FIFO维护一个访问计数器，它的值是访问同一个管道或FIFO的打开着的文件描述符个数。有了这个访问计数器后，客户或服务器就能成功地调用unlink. 尽管该函数从文件系统中删除了所指定的路径名，目前仍打开着的文件描述符却不受影响。 

---

管道和FIFO的数据是无边界的字节流（类似于TCP连接）。把这种字节流分隔成各个记录的任何方法都得由应用程序来实现。

**实现分隔的三种技巧：**
1. 带内特殊终止序列： 例如换行符\n, 例如网络应用FTP, SMTP, HTTP, NNTP用\r\n分隔。 
2. 显式长度：每个记录前冠以它的长度。 如 Sun RPC。
3. 每次连接一个记录：应用通过关闭与其对端的连接来指示一个记录的结束。 如 HTTP/1.0。 

#### 参考资料
- 《Unix 网络编程 卷二 2th》(UNP v2)
