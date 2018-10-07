---
layout:     post
title:      "在1G云主机上启动elasticsearch"
subtitle:   "在1核1G云主机上启动elasticsearch会出现一些错误..."
date:       2018-07-08
author:     "ChenWenKe"
tags:
    - 数据库 
---

## Elasticsearch简介
从数据存储的层面看：Elasticsearch是一种基于[RESTfull](https://en.wikipedia.org/wiki/Representational_state_transfer)接口的NoSQL分布式数据库，它提供全文搜索和结构化数据的实时统计，或者两者相结合。从搜索引擎的层面看：Elasticsearch是一个分布式，可扩展，实时的搜索与数据分析引擎。 

它可以快速的存储，搜索和分析海量数据。与MySQL一类的关系型数据库相比，它支持全文搜索，对象存储，实时统计，可以有效的使用数据。但是Elacsticsearch不支持ACID, 所以Elasticsearch不能完全保证数据的正确性。因此，Elasticsearch一般用于数据分析，数据统计和提高查询速度，作为二级数据存储而出现。例如：你大量的数据存储在腾讯云上，你搭建了一个网站来呈现这些数据，这时可以使用Elasticsearch作为中间数据库，使用Elasticsearch强大的搜索和分析能力，来方便数据的分析和统计，使用关系型数据库作为一级数据存储来保证数据的正确性。Elasticsearch使用非常广泛，维基百科，Stack Overflow, Github都在使用Elasticsearch做海量数据搜索和分析。


## 启动elasticsearch时出现的坑
Elasticsearch需要Java 8的环境，因此需要先安装 Java 8，然后下载解压Elasticsearch, `./bin/elasticsearch`启动即可。 但是启动时往往会出错：

### 1. Elasticsearch出于安全性考虑，不能使用 root 用户启动。
报错：
> [2018-07-09T13:06:16,832][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:125) ~[elasticsearch-6.2.4.jar:6.2.4]
	at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:112) ~[elasticsearch-6.2.4.jar:6.2.4]
	at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:86) ~[elasticsearch-6.2.4.jar:6.2.4]
	at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:124) ~[elasticsearch-cli-6.2.4.jar:6.2.4]
	at org.elasticsearch.cli.Command.main(Command.java:90) ~[elasticsearch-cli-6.2.4.jar:6.2.4]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:92) ~[elasticsearch-6.2.4.jar:6.2.4]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:85) ~[elasticsearch-6.2.4.jar:6.2.4]
Caused by: java.lang.RuntimeException: can not run elasticsearch as root
	at org.elasticsearch.bootstrap.Bootstrap.initializeNatives(Bootstrap.java:105) ~[elasticsearch-6.2.4.jar:6.2.4]
	at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:172) ~[elasticsearch-6.2.4.jar:6.2.4]
	at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:323) ~[elasticsearch-6.2.4.jar:6.2.4]
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:121) ~[elasticsearch-6.2.4.jar:6.2.4]
	... 6 more

解决方案： 用普通用户启动。
              
                                        
### 2. 需要赋予用户 Elasticsearch文件夹的权限，才能启动 Elasticsearch。
报错：
> Exception in thread "main" java.nio.file.AccessDeniedException: /usr/local/elasticsearch-6.2.4/config/jvm.options
	at sun.nio.fs.UnixException.translateToIOException(UnixException.java:84)
	at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:102)
	at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:107)
	at sun.nio.fs.UnixFileSystemProvider.newByteChannel(UnixFileSystemProvider.java:214)
	at java.nio.file.Files.newByteChannel(Files.java:361)
	at java.nio.file.Files.newByteChannel(Files.java:407)
	at java.nio.file.spi.FileSystemProvider.newInputStream(FileSystemProvider.java:384)
	at java.nio.file.Files.newInputStream(Files.java:152)
	at org.elasticsearch.tools.launchers.JvmOptionsParser.main(JvmOptionsParser.java:58)

解决方法：假设Elasticsearch 解压在elasticsearch-6.2.4文件夹里，`sudo chown -R <user>:<user> elasticsearch-6.2.4/`


### 3. elasticsearch-6.2.4启动时，默认需要初始化1G内存。当我使用1核1G的云主机启动Elasticsearch时会报如下错误： 
报错：
> OpenJDK 64-Bit Server VM warning: INFO: os::commit_memory(0x00000000c5330000, 986513408, 0) failed; error='Cannot allocate memory' (errno=12)
\#
\# There is insufficient memory for the Java Runtime Environment to continue.
\# Native memory allocation (mmap) failed to map 986513408 bytes for committing reserved memory.
\# An error report file with more information is saved as:
\# /usr/local/elasticsearch-6.2.4/hs_err_pid15738.log

解决方法：在`./config/jvm.options`文件中，把：
```
-Xms1g    # 初始化 JVM heap 为 1 G
-Xmx1g    # JVM heap最大为 1 G 
```
改为:
```
-Xms256m    # 初始化 JVM heap 为 256 M, 建议和Xmx大小相同。
-Xmx256m    # JVM heap最大为 256 M，建议不要超过主机物理内存的一半。
```


## elasticsearch入门教程

启动Elasticsearch, 检验是否成功。
```
curl -X GET 'http://localhost:9200/'
```

- [全文搜索引擎Elasticsearch入门教程](http://www.ruanyifeng.com/blog/2017/08/elasticsearch.html)
- [Elasticsearch:权威指南](http://elasticsearch.cn/book/elasticsearch_definitive_guide_2.x/index.html)




