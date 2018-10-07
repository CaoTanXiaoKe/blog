---
layout:     post
title:      "TCP 批量发送/接收结构体"
subtitle:   "为提高网络利用率，批量的发送/接收结构体"
date:       2017-07-15
author:     "ChenWenKe"
header-img: "img/post-bg-zhuan-liwei.jpg"
tags:
    - C++
    - 网络编程
---

#### TCP 批量发送/接收结构体

### 基本知识

**问题点：**  在项目中客户端需要向服务器端通过TCP发送很多的记录（固定大小的结构体）。客户端是单线程的，服务端是多线程的，即服务器端每次会创建一个线程跟客户端建立TCP连接来接收记录。 

- 正如我们所熟知的那样，TCP是字节流传输，所以接收的时候要处理一下不足值。 [关于TCP]()
- 每个TCP 分节（segment）都会带有 20 字节的IP头部， 至少20字节的 TCP头部， 所以如果每次只传送一个结构体对网络资源的利用率非常不好， 因此需要批量传输。 

### 代码

感觉自己这段代码写的很机智(好自恋)，而且网络编程中经常要处理TCP不足值的问题， 在此记录一下。

- **主要思路：** 客户端把要发送的结构体按二进制装进缓冲区，装满或者结构已装完时发送一个TCP segment. 服务端从TCP上接收数据，并截取成一个一个的结构体，如果剩余数据达不到一个结构体的长度，就继续接收。 思路类似于流水线。 细节处理比较多，详见代码。  

- 客户端发送代码

```cpp
void SocketSender::sendData(list<MLogRec>& logs) throw(SendException)
{
	 static const int BUFSIZE = 2048; 	// define buffer size， 套接字发送缓冲区低水位默认值即为：2048
	 static char buf[BUFSIZE]; 	  		// define buffer, 静态字符数组变量，防止栈溢出
	 int len = 0; 
	 int ret = 0; 
	 memset(buf, 0, sizeof(buf)); 
	 MLogRec mrec;
     while(logs.size())
     {
		if(len + sizeof(mrec) <= BUFSIZE)
		{
        	MLogRec mrec = logs.front();  
			logs.pop_front();  
			memcpy(buf+len, &mrec, sizeof(mrec)); 
			len += sizeof(mrec); 
		}
		else
		{
        	ret = send(sockfd, buf, len, 0); 
			len = 0; 
			if(ret == -1)
			{
				saveFailFile(logs); 
				throw SendException(); 
				break; 
			}
		}
     }
	 ret = send(sockfd, buf, len, 0); 
	 if(ret == -1)
	 {
		saveFailFile(logs); 
		throw SendException(); 
	 }
}
```

<br/>

- 服务器端接收代码

```cpp
// 接收客户端数据的线程处理函数, 并把数据放入存储队列
void ClientThread::run(void)
{
    extern LogQueue logQu; 
    list<MLogRec> logList; 
    MLogRec mrec; 
	const int BUFSIZE = 512;    // 相对小一些，防止栈溢出   
	char buf[BUFSIZE];          // Note: buf[] can't be global/static variable --- thread safe
	int rlen = 0; 
	int pos = 0; 
	int logLen = sizeof(mrec); 
	while((rlen = recv(m_accfd, buf, BUFSIZE, 0)) > 0)
	{
		int pos = 0; 
		while(rlen >= logLen) 
		{
			memcpy((char*)(&mrec) + (sizeof(mrec) - logLen), buf + pos, logLen); 
			pos += logLen; 
			rlen -= logLen; 
			
			logList.push_back(mrec); 
			logLen = sizeof(mrec); 
		}

		if(rlen < logLen)
		{
			memcpy((char*)(&mrec) + (sizeof(mrec) - logLen), buf + pos, rlen); 
			logLen -= rlen; 
		}
	}

    while(logList.size())
    {
        logQu << logList.front(); 
        logList.pop_front(); 
    }
}
``` 


> 备注：对于TCP和UDP而言，其套接字发送缓冲区低水位标记默认值通常就是 2048，我们可以使用 SO_SNDLOWAT套接字选项对其大小进行设置。 

<br/>
### 其他可考虑的替代方案：
- 添加分割符。适用于非二进制传输。 例如'\n'。
- 记录长度。“每条消息”头部添加固定大小的字节，用来记录“消息”的长度。 
- 每条连接是一条“消息”， HTTP/1.0中的方式。 

<br/>
<br/> 