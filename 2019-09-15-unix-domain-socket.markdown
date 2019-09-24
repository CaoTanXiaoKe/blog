---
layout:     post
title:      "Unix domain 套接字编程"
subtitle:   "以两个实例演示unix domain套接字的用法和用途"
date:       2019-09-15
author:     "ChenWenKe"
tags:
	- TCP/IP
	- 网络编程
mathjax: true

---



​        Unix domain套接字和TCP和UDP不同，主要用于单机间进程通信，可以看做是IPC中的一种。Unix套接字主要分成两类：字节流套接字(类似于TCP)， 数据报套接字(类似于UDP)。

​       使用Unix domain套接字主要有以下三个理由：

- 在同一个主机上进程间通信时，Unix domain套接字比TCP或UDP更加高效，而且所使用的接口和编程方式和TCP或UDP十分相似。
- 可以在进程间传递文件描述符(无论进程间是否有亲子关系)，结合setuid(设置用户ID)可以达到权限控制的目的。
-  Unix domain套接字较新的实现能够把client进程的凭证(用户ID和组ID)提供给服务器，从而能够提供额外的安全检查措施。

​        

​       本文主要关注第二点，即在进程间通过Unix domain (字节流)套接字进行文件描述符传递。从而达到用户A的进程，读取用户B的文件的目的。



####  示例 (一)：子进程传递描述符给父进程

类似于`cat`程序，我们编写一个`mycat`程序，它用于读取一个文件的内容打印到屏幕上。不同于`cat` 程序的是，`mycat`程序通过执行`fork`, `exec`一个设置用户ID的程序`openfile`获取要打开文件的文件描述符。

```bash
    mycat                                             openfile
+------------+                                   +----------------+
|            |         fork, exec                |                |
|  [0]       |      -------------------->        |            [1] |
+---+--------+                                   +-------------+--+
    |                                                          |
    |                         描述符                            |
    +----------------------------------------------------------+
```



##### mycat程序

```c
// mycat.c
#include	"unp.h"

int my_open(const char *, int);

int main(int argc, char **argv)
{
	int fd, n;
	char buff[BUFFSIZE];

	if (argc != 2)
		err_quit("usage: mycat <pathname>");

	if ( (fd = my_open(argv[1], O_RDONLY)) < 0)
		err_sys("cannot open %s", argv[1]);

	while ( (n = Read(fd, buff, BUFFSIZE)) > 0)
		Write(STDOUT_FILENO, buff, n);

	exit(0);
}

```

- [代码源自UNP](http://www.unpbook.com/src.html)

`mycat`程序通过调用`my_open`(类似于`open`接口，第一个参数为文件名，第二个参数为打开方式)获得要打开文件的描述符，然后通过文件描述符读取文件内容。

##### my_open函数

``` c
#include	"unp.h"

int my_open(const char *pathname, int mode)
{
	int	fd, sockfd[2], status;
	pid_t childpid;
	char c, argsockfd[10], argmode[10];

	Socketpair(AF_LOCAL, SOCK_STREAM, 0, sockfd);

	if ( (childpid = Fork()) == 0) {		/* child process */
		Close(sockfd[0]);
		snprintf(argsockfd, sizeof(argsockfd), "%d", sockfd[1]);
		snprintf(argmode, sizeof(argmode), "%d", mode);
		execl("./openfile", "openfile", argsockfd, pathname, argmode,
			  (char *) NULL);
		err_sys("execl error");
	}

	/* parent process - wait for the child to terminate */
	Close(sockfd[1]);			/* close the end we don't use */

	Waitpid(childpid, &status, 0);
	if (WIFEXITED(status) == 0)
		err_quit("child did not terminate");
	if ( (status = WEXITSTATUS(status)) == 0)
		Read_fd(sockfd[0], &c, 1, &fd);
	else {
		errno = status;		/* set errno value from child's status */
		fd = -1;
	}

	Close(sockfd[0]);
	return(fd);
}

```

`my_open`函数通过fork子进程执行`openfile`打开文件，并且通过`Read_fd(sockfd[0], &c, 1, &fd)`读取到`openfile`打开文件所获得的文件描述符。

##### openfile程序

```c
// openfile.c
#include	"unp.h"

int main(int argc, char **argv)
{
	int fd;
  
	if (argc != 4)
		err_quit("openfile <sockfd#> <filename> <mode>");

	if ( (fd = open(argv[2], atoi(argv[3]))) < 0)
		exit( (errno > 0) ? errno : 255 );

	if (write_fd(atoi(argv[1]), "", 1, fd) < 0)
		exit( (errno > 0) ? errno : 255 );

	exit(0);
}
```



##### read_fd函数和write_fd函数

```c
/* include read_fd */
#include	"unp.h"

ssize_t read_fd(int fd, void *ptr, size_t nbytes, int *recvfd)
{
	struct msghdr	msg;
	struct iovec	iov[1];
	ssize_t			n;

#ifdef	HAVE_MSGHDR_MSG_CONTROL
	union {
	  struct cmsghdr	cm;
	  char				control[CMSG_SPACE(sizeof(int))];
	} control_un;
	struct cmsghdr	*cmptr;

	msg.msg_control = control_un.control;
	msg.msg_controllen = sizeof(control_un.control);
#else
	int				newfd;

	msg.msg_accrights = (caddr_t) &newfd;
	msg.msg_accrightslen = sizeof(int);
#endif

	msg.msg_name = NULL;
	msg.msg_namelen = 0;

	iov[0].iov_base = ptr;
	iov[0].iov_len = nbytes;
	msg.msg_iov = iov;
	msg.msg_iovlen = 1;

	if ( (n = recvmsg(fd, &msg, 0)) <= 0)
		return(n);

#ifdef	HAVE_MSGHDR_MSG_CONTROL
	if ( (cmptr = CMSG_FIRSTHDR(&msg)) != NULL &&
	    cmptr->cmsg_len == CMSG_LEN(sizeof(int))) {
		if (cmptr->cmsg_level != SOL_SOCKET)
			err_quit("control level != SOL_SOCKET");
		if (cmptr->cmsg_type != SCM_RIGHTS)
			err_quit("control type != SCM_RIGHTS");
		*recvfd = *((int *) CMSG_DATA(cmptr));
	} else
		*recvfd = -1;		/* descriptor was not passed */
#else
/* *INDENT-OFF* */
	if (msg.msg_accrightslen == sizeof(int))
		*recvfd = newfd;
	else
		*recvfd = -1;		/* descriptor was not passed */
/* *INDENT-ON* */
#endif

	return(n);
}
/* end read_fd */

ssize_t Read_fd(int fd, void *ptr, size_t nbytes, int *recvfd)
{
	ssize_t		n;

	if ( (n = read_fd(fd, ptr, nbytes, recvfd)) < 0)
		err_sys("read_fd error");

	return(n);
}
```

```c
/* include write_fd */
#include	"unp.h"

ssize_t write_fd(int fd, void *ptr, size_t nbytes, int sendfd)
{
	struct msghdr	msg;
	struct iovec	iov[1];

#ifdef	HAVE_MSGHDR_MSG_CONTROL
	union {
	  struct cmsghdr	cm;
	  char				control[CMSG_SPACE(sizeof(int))];
	} control_un;
	struct cmsghdr	*cmptr;

	msg.msg_control = control_un.control;
	msg.msg_controllen = sizeof(control_un.control);

	cmptr = CMSG_FIRSTHDR(&msg);
	cmptr->cmsg_len = CMSG_LEN(sizeof(int));
	cmptr->cmsg_level = SOL_SOCKET;
	cmptr->cmsg_type = SCM_RIGHTS;
	*((int *) CMSG_DATA(cmptr)) = sendfd;
#else
	msg.msg_accrights = (caddr_t) &sendfd;
	msg.msg_accrightslen = sizeof(int);
#endif

	msg.msg_name = NULL;
	msg.msg_namelen = 0;

	iov[0].iov_base = ptr;
	iov[0].iov_len = nbytes;
	msg.msg_iov = iov;
	msg.msg_iovlen = 1;

	return(sendmsg(fd, &msg, 0));
}
/* end write_fd */

ssize_t Write_fd(int fd, void *ptr, size_t nbytes, int sendfd)
{
	ssize_t		n;

	if ( (n = write_fd(fd, ptr, nbytes, sendfd)) < 0)
		err_sys("write_fd error");

	return(n);
}

```



##### 执行

参照unp代码的编译方式编译得到两个可执行文件，**mycat**和**openfile**。

```bash
# 改变openfile的拥有者，使得openfile属于root用户。
$ sudo chown root:root openfile
$
# 创建文件 IF.txt 并写入一些内容
# 改变文件 IF.txt 的读写权限，使得IF.txt只能由 root用户读取
$ sudo chmod 600 IF.txt
$
# 查看各文件状态如下：
$ ls -l mycat openfile IF.txt
-rw------- 1 root   root     371 Sep 15 16:42 IF.txt
-rwxrwxr-x 1 ubuntu ubuntu 68304 Sep 15 16:37 mycat
-rwxrwxr-x 1 root   root   24784 Sep 15 16:37 openfile
$
$
# 此时在ubuntu用户下，使用mycat读取IF.txt文件内容失败
$ ./mycat IF.txt
cannot open IF.txt: Permission denied
$
# 对可执行文件 openfile 执行设置用户ID操作
$ sudo chmod u+s openfile
$ ls -l openfile
-rwsrwxr-x 1 root root 24784 Sep 15 16:37 openfile
$
# 此时再次使用 mycat 读取IF.txt文件的内容
$ ./mycat IF.txt
If you can keep your head when all about you
Are losing theirs and blaming it on you;
I f you can trust yourself when all men doubt you,
But make allowance for their doubting too;
If you can wait and not be tired by waiting,
Or, being lied about, don’t deal in lies,
Or, being hated, don’t give away to hating,
And yet don’t look too good, nor talk too wise;

...
```



以上，实现了在用户ubuntu下，读取了只允许root权限读取的IF.txt文件的内容。以进程间传递文件描述符的方式，给予了用户ubuntu操作root权限下的文件相当大的自由度，同时也可以对openfile程序做一定的修改从而达到更细粒度的对用户ubuntu的操作进行控制的目的。



#### 示例(二)：没有亲缘关系的两个进程间传递描述符

对上面的程序稍做修改，使得描述符的传递跨不具有亲缘关系的两个进程， openfile_server程序类似于openfile程序，提供一个Unix domain 字节流服务用于传递打开文件的描述符。myopen_client 向openfile_server发起连接，并从建立的连接上获取打开文件的描述符。

```bash
    mycat                                           openfile_server
+------------+                                   +-----------------------+
|            |              connect              |                       |
| [sockfd]   |      -------------------->        | [listenfd]   [connfd] |
+---+--------+                                   +------+---------- +----+
    |                                                               |
    |                         描述符                                 |
    +---------------------------------------------------------------+
```



##### openfile_server程序

```c
// openfile_server.c
#include "unp.h"

int main(int argc, char **argv)
{
    int listenfd, connfd;
    struct sockaddr_un servaddr, cliaddr;
    socklen_t clilen;
    char buf[MAXLINE];

    listenfd = Socket(AF_LOCAL, SOCK_STREAM, 0);
    // use unlink to delete UNIXSTR_PATH 
    unlink(UNIXSTR_PATH);       // UNIXSTR_PATH: tmp/unix.str defined in unp.h
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sun_family = AF_LOCAL;
    strcpy(servaddr.sun_path, UNIXSTR_PATH);
    Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));
    Listen(listenfd, LISTENQ);
    while(1)
    {
        if ( (connfd = accept(listenfd, (SA *) &cliaddr, &clilen)) < 0)
        {
            if (errno == EINTR)
                continue;   // back to while()
            else
                err_sys("accept error");
        }
        
        int nread;
        if ( (nread = Readline(connfd, buf, MAXLINE)) < 0)
            err_sys("read from connfd failed");
        char filename[FILENAME_MAX];
        int mode;
        sscanf(buf, "%s %d \n", filename, &mode);

        int fd;
        if ( (fd = open(filename, mode)) < 0)
            err_sys("open file failed");

        if (write_fd(connfd, "", 1, fd) < 0)
            err_sys("write_fd failed");
        
        Close(fd);  // remember close fd, because sendmsg would make 
                    // reference of fd count increase one.
        Close(connfd);  
    }
    Close(listenfd);

    exit(0);
}
```





##### myopen_client程序

```c
// myopen_client.c
#include "unp.h"

int my_open(const char *pathname, int mode)
{
    int sockfd, fd;
    struct sockaddr_un servaddr;
    char buf[MAXLINE];

    sockfd = Socket(AF_LOCAL, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sun_family = AF_LOCAL;
    strcpy(servaddr.sun_path, UNIXSTR_PATH);
    Connect(sockfd, (SA *) &servaddr, sizeof(servaddr));
    // pathname mode \n
    sprintf(buf, "%s %d \n%c", pathname, mode, '\0');
    Writen(sockfd, buf, strlen(buf));
    char c;
    Read_fd(sockfd, &c, 1, &fd);
    Close(sockfd);
    return fd;
}
```







##### 编译 & 执行

```bash
# 编译 openfile_server 程序
$ gcc -I../lib -g -O2 -D_REENTRANT -Wall   -c -o openfile_server.o openfile_server.c
$ gcc -I../lib -g -O2 -D_REENTRANT -Wall -o openfile_server openfile_server.o ../libunp.a -lresolv -lpthread
# 得到可执行程序 openfile_server
$
# 编译 mycat 程序(myopen函数使用的是myopen_client.c文件里的实现)
$ gcc -I../lib -g -O2 -D_REENTRANT -Wall   -c -o mycat.o mycat.c                
$ gcc -I../lib -g -O2 -D_REENTRANT -Wall   -c -o myopen_client.o myopen_client.c
$ gcc -I../lib -g -O2 -D_REENTRANT -Wall -o mycat mycat.o myopen_client.o ../libunp.a -lresolv -lpthread
$
# 执行 ~~~
# 参考示例 < 一 >
$
```



#### 知识点

- Unix domain 所绑定的路径，和TCP/UDP中的 `<IP>:<Port>`的作用类似, **注意：路径名尽量使用绝对路径, 最长不要超过104字节**。
- Unix domain 所绑定的路径尽量使用绝对路径，总长度不要超过104(包括结尾NULL)字节。
- `socketpair`  函数会创建两个未命名的套接字(也就是说，在这两个套接字上没有隐式的bind调用)，调用`socketpair`创建的结果是一个流管道，类似于调用pipe创建的普通管道，差别在于流管道是全双工的。
- `bind`函数调用前，文件系统中如果已存在要绑定的路径，将会绑定失败(因此程序中我们需要先调用`unlink`函数，删除该路径)。
- 调用`connect`连接一个Unix domain套接字涉及的权限测试等同于调用`open`以只写方式访问相应的路径名。不同于TCP/UDP，如果没有绑定IP和端口，在connect时会为套接字选择一个IP和一个随机端口， Unix domain套接字在connect时，如果没有绑定路径，不会自动为该套接字绑定一个路径。
- 如果对于某个Unix domain字节流套接字的connect调用，发现这个监听套接字的队列已满，调用会立即返回一个ECONNREFUSER错误。这一点不同于TCP，如果TCP的监听套接字已满，TCP监听端会忽略新到达的SYN分节，而TCP连接发起端将进行退避重试。
- 使用`sendmsg`的辅助数据传递描述符时，会使该描述符的**引用计数加一**， 这样在发送进程调用`sendmsg`之后，接收进程调用`recvmsg`之前(即：这个描述符“在飞行中(in flight)”)。发送进程此时关闭该描述符也没有什么问题。
- 接收进程调用`recvmsg`接收到的描述符数值和发送进程中描述符数值不一定相同。用Unix domain套接字传递描述符并不是传递文件描述符数值，而是涉及到由内核在接收进程中创建一个新的文件描述符，而这个新的文件描述符和发送进程中的那个描述符指向内核中相同的文件表项。



#### 总结

​        本文展示了两个Unix domain套接字编程(字节流)的示例，其中第一个示例来自《Unix 网络编程》(第三卷)， 第二个示例是在第一个示例上略做了一点儿修改。这两个示例涵盖了Unix domain套接字编程的大部分核心知识点和注意事项，希望能对你有所帮助。





PS: 很多优秀的书籍中的例程，在实际工程和学习中大有裨益。当我们掌握了核心知识点，并且有例程 by hand时，世界真的会变得美好很多。





> 好的设计一定是在精通的基础上，而不是从无知中产生的。 
>
> ​                                                                 — 《设计原本》 Brooks









