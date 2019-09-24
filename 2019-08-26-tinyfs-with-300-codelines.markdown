---
layout:     post
title:      "「转」300行代码实现Linux文件系统"
subtitle:   "300来行代码带你实现一个能跑的最小Linux文件系统"
date:       2019-08-26
author:     "ChenWenKe"
tags:
  - OS
  - Linux
mathjax: true

---

### 缘起

从微信公众号上转载一篇有趣的文章: [300来行代码带你实现一个能跑的最小Linux文件系统](https://mp.weixin.qq.com/s/Sidfn8CZn4KxKh6xMH2uJQ)。 通过这边文章，可以实例演练一下上一篇[编译加载Linux内核模块](https://www.chenwenke.cn/blog/2019/08/25/2019-08-25-complie-kernel-module/), 进而加深对Linux内核模块的理解。

---



### 代码

由于微信公众号上代码排版有些奇特，文章中的代码整理如下(CentOS下可编译运行)：

```c
// tinyfs.h
#define MAXLINE     8
#define MAX_FILES   32
#define MAX_BLOCKSIZE   512

// 定义每一个目录项的格式
struct dir_entry {
    char filename[MAXLINE];
    uint8_t idx;
};

// 定义每一个文件的格式。
struct file_blk {
    uint8_t busy;
    mode_t mode;
    uint8_t idx;

    union {
        uint8_t file_size;
        uint8_t dir_children;
    };
    char data[0];
};
```



```c
// tinyfs.c

#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include "tinyfs.h"


struct file_blk block[MAX_FILES+1];
int curr_count = 0;


// 获得一个尚未使用的文件块，保存新创建的文件或者目录
static int get_block(void)
{
    int i;
    // 就是一个遍历，但实现快速。
    for (i = 2; i < MAX_FILES; i++) {
        if (!block[i].busy) {
            block[i].busy = 1;
            return i;
        }
    }
    return -1;
}

static struct inode_operations tinyfs_inode_ops;

// 读取目录的实现
static int tinyfs_readdir(struct file *filp, void *dirent, filldir_t filldir)
{
    loff_t pos;
    struct file_blk *blk;
    struct dir_entry *entry;
    int i;

    pos = filp->f_pos;
    if (pos)
        return 0;

    blk = (struct file_blk *)filp->f_dentry->d_inode->i_private;

    if (!S_ISDIR(blk->mode)) {
        return -ENOTDIR;
    }
    
    // 循环获取一个目录的所有文件的文件名
    entry = (struct dir_entry *)&blk->data[0];
    for (i = 0; i < blk->dir_children; i++) {
        filldir(dirent, entry[i].filename, MAXLINE, pos, entry[i].idx, DT_UNKNOWN);
        filp->f_pos += sizeof(struct dir_entry);
        pos += sizeof(struct dir_entry);
    }

    return 0;
}

// read实现
ssize_t tinyfs_read(struct file * filp, char __user * buf, size_t len, loff_t *ppos)
{
    struct file_blk *blk;
    char *buffer;
    blk = (struct file_blk *)filp->f_path.dentry->d_inode->i_private;

    if (*ppos >= blk->file_size)
        return 0;

    buffer = (char *)&blk->data[0];
    len = min((size_t) blk->file_size, len);

    if (copy_to_user(buf, buffer, len)) {
        return -EFAULT;
    }

    *ppos += len;
    return len;
}

// write实现
ssize_t tinyfs_write(struct file * filp, const char __user * buf, size_t len, loff_t * ppos)
{
    struct file_blk *blk;
    char *buffer;
    blk = filp->f_path.dentry->d_inode->i_private;
    buffer = (char *)&blk->data[0];

    buffer += *ppos;
    if (copy_from_user(buffer, buf, len)) {
        return -EFAULT;
    }

    *ppos += len;
    blk->file_size = *ppos;
    return len;
}


const struct file_operations tinyfs_file_operations = {
    .read = tinyfs_read,
    .write = tinyfs_write,
};

const struct file_operations tinyfs_dir_operations = {
    .owner = THIS_MODULE,
    .readdir = tinyfs_readdir,
};

// 创建文件的实现
static int tinyfs_do_create(struct inode *dir, struct dentry *dentry, umode_t mode)
{

    struct inode *inode;
    struct super_block *sb;
    struct dir_entry *entry;
    struct file_blk *blk, *pblk;
    int idx;
    sb = dir->i_sb;

    if (curr_count >= MAX_FILES) {
        return -ENOSPC;
    }

    if (!S_ISDIR(mode) && !S_ISREG(mode)) {
        return -EINVAL;
    }

    inode = new_inode(sb);
    if (!inode) {
        return -ENOMEM;
    }
    inode->i_sb = sb;
    inode->i_op = &tinyfs_inode_ops;
    inode->i_atime = inode->i_mtime = inode->i_ctime = CURRENT_TIME;

    idx = get_block(); // 获取一个空闲的文件块保存新文件
    blk = &block[idx];
    inode->i_ino = idx;
    blk->mode = mode;

    curr_count ++;

    if (S_ISDIR(mode)) {
        blk->dir_children = 0;
        inode->i_fop = &tinyfs_dir_operations;
    } else if (S_ISREG(mode)) {
        blk->file_size = 0;
        inode->i_fop = &tinyfs_file_operations;
    }

    inode->i_private = blk;
    pblk = (struct file_blk *)dir->i_private;
    entry = (struct dir_entry *)&pblk->data[0];
    entry += pblk->dir_children;
    pblk->dir_children ++;
    entry->idx = idx;
    strcpy(entry->filename, dentry->d_name.name);

    // VFS穿针引线的关键步骤，将VFS的inode链接到链表
    inode_init_owner(inode, dir, mode);
    d_add(dentry, inode);
    return 0;
}

static int tinyfs_mkdir(struct inode *dir, struct dentry *dentry, umode_t mode)
{
    return tinyfs_do_create(dir, dentry, S_IFDIR | mode);
}

static int tinyfs_create(struct inode *dir, struct dentry *dentry, umode_t mode, bool excl)
{
    return tinyfs_do_create(dir, dentry, mode);
}

static struct inode *tinyfs_iget(struct super_block *sb, int idx)
{
    struct inode *inode;
    struct file_blk *blk;

    inode = new_inode(sb);
    inode->i_ino = idx;
    inode->i_sb = sb;
    inode->i_op = &tinyfs_inode_ops;
    blk = &block[idx];

    if (S_ISDIR(blk->mode))
        inode->i_fop = &tinyfs_dir_operations;
    else if (S_ISREG(blk->mode))
        inode->i_fop = &tinyfs_file_operations;

    inode->i_atime = inode->i_mtime = inode->i_ctime = CURRENT_TIME;
    inode->i_private = blk;
    return inode;
}

struct dentry *tinyfs_lookup(struct inode *parent_inode, struct dentry *child_dentry, unsigned int flags)
{
    struct super_block *sb = parent_inode->i_sb;
    struct file_blk *blk;
    struct dir_entry *entry;
    int i;
    blk = (struct file_blk *)parent_inode->i_private;
    entry = (struct dir_entry *)&blk->data[0];
    for (i = 0; i < blk->dir_children; i++) {
        if (!strcmp(entry[i].filename, child_dentry->d_name.name)) {
            struct inode *inode = tinyfs_iget(sb, entry[i].idx);
            struct file_blk *inner = (struct file_blk *)inode->i_private;
            inode_init_owner(inode, parent_inode, inner->mode);
            d_add(child_dentry, inode);
            return NULL;
        }
    }

    return NULL;
}

int tinyfs_rmdir(struct inode *dir, struct dentry *dentry)
{
    struct inode *inode = dentry->d_inode;
    struct file_blk *blk = (struct file_blk *)inode->i_private;
    blk->busy = 0;
    return simple_rmdir(dir, dentry);
}

int tinyfs_unlink(struct inode *dir, struct dentry *dentry)
{
    int i;
    struct inode *inode = dentry->d_inode;
    struct file_blk *blk = (struct file_blk *)inode->i_private;
    struct file_blk *pblk = (struct file_blk *)dir->i_private;
    struct dir_entry *entry;

    // 更新其上层目录
    entry = (struct dir_entry *)&pblk->data[0];
    for (i = 0; i < pblk->dir_children; i++) {
        if (!strcmp(entry[i].filename, dentry->d_name.name)) {
            int j;
            for (j = i; j < pblk->dir_children - 1; j++) {
                memcpy(&entry[j], &entry[j+1], sizeof(struct dir_entry));
            }
            pblk->dir_children --;
            break;
        }
    }

    blk->busy = 0;
    return simple_unlink(dir, dentry);
}

static struct inode_operations tinyfs_inode_ops = {
    .create = tinyfs_create,
    .lookup = tinyfs_lookup,
    .mkdir = tinyfs_mkdir,
    .rmdir = tinyfs_rmdir,
    .unlink = tinyfs_unlink,
};

int tinyfs_fill_super(struct super_block *sb, void *data, int silent)
{
    struct inode *root_inode;
    int mode = S_IFDIR;
    root_inode = new_inode(sb);
    root_inode->i_ino = 1;
    inode_init_owner(root_inode, NULL, mode);
    root_inode->i_sb = sb;
    root_inode->i_op = &tinyfs_inode_ops;
    root_inode->i_fop = &tinyfs_dir_operations;
    root_inode->i_atime = root_inode->i_mtime = root_inode->i_ctime = CURRENT_TIME;

    block[1].mode = mode;
    block[1].dir_children = 0;
    block[1].idx = 1;
    block[1].busy = 1;
    root_inode->i_private = &block[1];

    sb->s_root = d_make_root(root_inode);
    curr_count++;

    return 0;
}

static struct dentry *tinyfs_mount(struct file_system_type *fs_type, int flags, const char *dev_name, void *data)
{
    return mount_nodev(fs_type, flags, data, tinyfs_fill_super);
}

static void tinyfs_kill_superblock(struct super_block *sb)
{
    kill_anon_super(sb);
}

struct file_system_type tinyfs_fs_type = {
    .owner = THIS_MODULE,
    .name = "tinyfs",
    .mount = tinyfs_mount,
    .kill_sb = tinyfs_kill_superblock,
};

static int tinyfs_init(void)
{
    int ret;
    memset(block, 0, sizeof(block));
    ret = register_filesystem(&tinyfs_fs_type);
    if (ret)
    printk("register tinyfs failed\n");
    return ret;
}

static void tinyfs_exit(void)
{
    unregister_filesystem(&tinyfs_fs_type);
}

module_init(tinyfs_init);
module_exit(tinyfs_exit);
MODULE_LICENSE("GPL");
```



```makefile
# Makefile
# CONFIG_MODULE_SIG=n

obj-m += tinyfs.o
KERNELDIR ?= /lib/modules/$(shell uname -r)/build
all:
        make -C $(KERNELDIR) M=$(PWD) modules
clean:
        make -C $(KERNELDIR) M=$(PWD) clean
```





### 编译&运行

```bash
# ls
Makefile  tinyfs.c  tinyfs.h
#
# make
#
# ls
Makefile  modules.order  Module.symvers  tinyfs.c  tinyfs.h  tinyfs.ko  tinyfs.mod.c  tinyfs.mod.o  tinyfs.o
# # 加载内核模块： 文件系统ko， tinyfs.ko
# insmod tinyfs.ko
#
# # 挂在文件系统 tinyfs 到 /mnt 目录下
# ls /mnt/
#
# mount -t tinyfs none /mnt
#
# ls /mnt/
#
# cd /mnt
# 
# echo 11111 > ./a
#
# cat ./a
11111
#
# mkdir dir1
# echo 22222 > ./dir1/b
#
# cat ./dir1/b
22222
#
# tree
.
├── a
└── dir1
    └── b

1 directory, 2 files
#
# echo 333 > ./dir1/c
# 
# cat ./dir1/c
333
#
# tree
.
├── a
└── dir1
    ├── b
    └── c

1 directory, 3 files
#
# rm -f ./dir1/c
#
# tree
.
├── a
└── dir1
    └── b

1 directory, 2 files
#
# rm -rf dir1/
#
# tree
.
├── a
└── dir1

1 directory, 1 file
#
# rm -rf dir1/
#
# # 因为我们没有实现删除目录的接口，因此目录无法删除。
# tree
.
├── a
└── dir1

1 directory, 1 file
#
#
#
# # 卸载文件系统 tinyfs文件系统
# cd ~
# umount -f /mnt/
#
# # 卸载tinyfs内核模块
# rmmod tinyfs
#
#
```

