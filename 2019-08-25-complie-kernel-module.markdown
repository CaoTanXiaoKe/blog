---
layout:     post
title:      "编译加载Linux内核模块"
subtitle:   "编译和加载一个简单的Linux内核模块，演示编译和加载内核模块的过程"
date:       2019-08-25
author:     "ChenWenKe"
tags:
  - OS
  - Linux
mathjax: true

---



### 工具

- `depmod`: 生成描述模块之间的依赖关系，写入到相关文件(可以通过`man depmod`查看详情)。
- `insmod`: 加载*.ko的内核模块到内核中，类似于用户空间使用的动态库加载程序。注意，如果加载的模块有依赖模块，需要先`insmod <依赖模块>`。
- `rmmod` 卸载模块。
- `lsmod`: 查看系统中已经加载了的模块。
- `modinfo`: 查看*.ko 内核模块的信息。
- `modprobe`:  默认读取相关文件记录的模块，查看相关依赖，然后依次调用`insmod`加载各个模块。`modprobe`相当于`insmod`的智能版(通过`man modprobe`查看详情)。



### 一个简单的内核模块 — hello world

```c
/*  
 *  hello-4.c - Demonstrates module documentation.
 */
#include <linux/module.h>	/* Needed by all modules */
#include <linux/kernel.h>	/* Needed for KERN_INFO */
#include <linux/init.h>		/* Needed for the macros */
#define DRIVER_AUTHOR "Peter Jay Salzman <p@dirac.org>"
#define DRIVER_DESC   "A sample driver"

static int __init init_hello_4(void)
{
	printk(KERN_INFO "Hello, world 4\n");
	return 0;
}

static void __exit cleanup_hello_4(void)
{
	printk(KERN_INFO "Goodbye, world 4\n");
}

module_init(init_hello_4);
module_exit(cleanup_hello_4);

/*  
 *  You can use strings, like this:
 */

/* 
 * Get rid of taint message by declaring code as GPL. 
 */
MODULE_LICENSE("GPL");

/*
 * Or with defines, like this:
 */
MODULE_AUTHOR(DRIVER_AUTHOR);	/* Who wrote this module? */
MODULE_DESCRIPTION(DRIVER_DESC);	/* What does this module do */

/*  
 *  This module uses /dev/testdevice.  The MODULE_SUPPORTED_DEVICE macro might
 *  be used in the future to help automatic configuration of modules, but is 
 *  currently unused other than for documentation purposes.
 */
MODULE_SUPPORTED_DEVICE("testdevice");
```



每个内核模块都包含一个注册函数(`module_init(init_hello_4);`)和一个清理函数`module_exit(cleanup_hello_4);`。在模块加载的时候会调用注册函数，在模块卸载时会调用清理函数。

内核模块编写的一般套路是，在注册函数内，往内核模块的钩子上，注册上自己实现的函数(往往是先做一些自定义的操作，然后调用原来的内核函数)，从而达到改变内核行为的目的。而清理模块则相当于注册函数的析构，做一些清理和还原的操作。(PS： Nginx里的模块开发也是使用钩子这种操作。)

上面的`hello-4.c`的代码逻辑十分简单，在注册函数里`printk(KERN_INFO "Hello, world 4\n");` 即打印字符串`Hello, world 4`到系统日志文件(centos里为 `/var/log/message`, ubuntu里为`/var/log/syslog`)，`KERN_INFO`指定日志级别。`MODULE_AUTHOR`， `MODULE_DESCRIPTION` 用于添加对该内核模块的描述信息，这些信息可以通过`modinfo`进行查看。



### 编译内核模块

- 编写Makefile

  ```make
  # CONFIG_MODULE_SIG=n
  
  obj-m += hello-4.o
  
  KERNELDIR ?= /lib/modules/$(shell uname -r)/build
  all:
  	make -C $(KERNELDIR) M=$(PWD) modules
  
  clean:
  	make -C $(KERNELDIR) M=$(PWD) clean
  ```

  - `# CONFIG_MODULE_SIG=n` 用于关闭签名验证，因为目前大多数的内核发行版都会启用签名，来检查加载的是否是它验证过的内核模块。
  - `obj-m` 是编写内核makefile的语法，表示目标文件是一个内核模块。[详情查看文档](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt)
  - `shell uname -r`是查看本机的内核版本号； `/lib/modules/$(shell uname -r)/build`目录下存放着部分编译该机器内核时所使用的源码。包括：`<linux/module.h>` , `<linux/kernel.h>`, `<linux/init.h>`等。

- 编译：`make`

  ```bash
  [root@VM_1_62_centos Test]# make
  make -C /lib/modules/3.10.0-957.21.3.el7.x86_64/build M=/root/WorkSpace/scf_vpc/src/Test modules
  make[1]: 进入目录“/usr/src/kernels/3.10.0-957.21.3.el7.x86_64”
    CC [M]  /root/WorkSpace/scf_vpc/src/Test/hello-4.o
    Building modules, stage 2.
    MODPOST 1 modules
    CC      /root/WorkSpace/scf_vpc/src/Test/hello-4.mod.o
    LD [M]  /root/WorkSpace/scf_vpc/src/Test/hello-4.ko
  make[1]: 离开目录“/usr/src/kernels/3.10.0-957.21.3.el7.x86_64”
  [root@VM_1_62_centos Test]# 
  [root@VM_1_62_centos Test]# ls
  hello-4.c   hello-4.mod.c  hello-4.o  modules.order
  hello-4.ko  hello-4.mod.o  Makefile   Module.symvers
  [root@VM_1_62_centos Test]# 
  [root@VM_1_62_centos Test]#
  ```

- 查看模块信息

  ```bash
  [root@VM_1_62_centos Test]# modinfo hello-4.ko 
  filename:       /root/WorkSpace/scf_vpc/src/Test/hello-4.ko
  description:    A sample driver
  author:         Peter Jay Salzman <p@dirac.org>
  license:        GPL
  retpoline:      Y
  rhelversion:    7.6
  srcversion:     9ACA98AD11775A2DCB1EE86
  depends:        
  vermagic:       3.10.0-957.21.3.el7.x86_64 SMP mod_unload modversions 
  [root@VM_1_62_centos Test]# 
  [root@VM_1_62_centos Test]# 
  ```

### 加载内核模块

由上面使用`modinfo`查看的信息，可以看到我们的模块没有依赖其它的模块。可以直接使用`insmod`进行动态加载:

```bash
[root@VM_1_62_centos Test]# 
[root@VM_1_62_centos Test]# insmod hello-4.ko 
[root@VM_1_62_centos Test]# 
[root@VM_1_62_centos Test]# lsmod | grep "hello"
hello_4                12430  0 
[root@VM_1_62_centos Test]# 
```

查看`/var/log/message` （CentOS）可以看到如下日志:

```vim
Aug 25 01:06:24 VM_1_62_centos kernel: Hello, world 4
```

卸载模块：

```bash
[root@VM_1_62_centos Test]# 
[root@VM_1_62_centos Test]# rmmod hello-4.ko 
[root@VM_1_62_centos Test]# 
[root@VM_1_62_centos Test]# lsmod | grep "hello"
[root@VM_1_62_centos Test]# 
```

在`/var/log/message`中看到如下日志：

```vim
Aug 25 01:09:41 VM_1_62_centos kernel: Goodbye, world 4
```




>后记： 万事开头难，然后中间往往无限漫长，因此没有结尾。嗯，这便是修行。



### 参考资料

- [The Linux Kernel Module Programming Guide](https://www.tldp.org/LDP/lkmpg/2.6/html/x279.html)

