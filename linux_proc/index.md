# 内核空间和用户文件通信 - proc 虚拟文件系统


## 概述 {#概述}

使用 proc 虚拟文件系统来在内核空间和用户空间交换数据应该是最常用的一种方式了。我们可以在 `/proc` 目录下创建一个虚拟的文件，通过读写这个文件实现用户空间和内核空间的数据交互。

**需要说明的是，虽然这里我们只讨论 procfs，但是在在新一些的 linux kernel 中我们应该使用 sysfs 来替代 procfs 作为 driver 和 user space 的接口。**


## 代码示例 {#代码示例}

废话不多说，先上例子。（例子是基于 linux 3.9.9 测试的，但是理论上适用于到目前为止的所有 kernel 版本）

```c
/*
 * procfs1.c
 */

#include <linux/kernel.h>		/* We're doing kernel work */
#include <linux/module.h>		/* Specifically, a module */
#include <linux/proc_fs.h>		/* Necessary because we use the proc fs */
#include <linux/uaccess.h>		/* for copy_from_user */
#include <linux/version.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
#define HAVE_PROC_OPS
#endif

#if LINUX_VERSION_CODE < KERNEL_VERSION(3, 10, 0)
#define HAVE_PROC_READ_SUITE
#endif

#define PROCFS_MAX_SIZE		1024
#define PROCFS_NAME			"pbuf"

/* This structure hold information about the /proc file */
static struct proc_dir_entry *our_proc_file;

/* The buffer used to store character for this module */
static char procfs_buffer[PROCFS_MAX_SIZE];
/* The size of the buffer */
static unsigned long procfs_buffer_size = 0;

static ssize_t procfile_read(struct file *file, const char __user *buff,
                             size_t len, loff_t *off)
{
	int plen = procfs_buffer_size;
    ssize_t ret = plen;

    if (*off >= plen || copy_to_user(buff, procfs_buffer, plen)) {
        pr_info("copy_to_user failed\n");
        ret = 0;
    } else {
        pr_info("procfile read %s\n", file->f_path.dentry->d_name.name);
        *off += plen;
    }

    return ret;
}

/*
cat /dev/zero | tr "\0" "a" | dd of=/tmp/1.txt bs=1023 count=1
cat /tmp/1.txt > /proc/pbuf
*/
static ssize_t procfile_write(struct file *file, const char __user *buff,
                              size_t len, loff_t *off)
{
    procfs_buffer_size = len;
    if (procfs_buffer_size > PROCFS_MAX_SIZE)
        procfs_buffer_size = PROCFS_MAX_SIZE;

	if (copy_from_user(procfs_buffer, buff, procfs_buffer_size)) {
		pr_info("copy_from_user failed\n");
        return -EFAULT;
	}

	/* supports only max to 1023 bytes string */
	procfs_buffer[procfs_buffer_size & (PROCFS_MAX_SIZE - 1)] = '\0';
    pr_info("procfile write %s\n", procfs_buffer);

    return procfs_buffer_size;
}

#ifdef HAVE_PROC_OPS
static const struct proc_ops proc_file_fops = {
    .proc_read = procfile_read,
	.proc_write = procfile_write,
};
#else
static const struct file_operations proc_file_fops = {
    .read = procfile_read,
	.write = procfile_write,
};
#endif

static int __init procfs1_init(void)
{
    our_proc_file = proc_create(PROCFS_NAME, 0644, NULL, &proc_file_fops);
    if (NULL == our_proc_file) {
#ifdef HAVE_PROC_READ_SUITE
		remove_proc_entry(PROCFS_NAME, NULL);
#else
        proc_remove(our_proc_file);
#endif
        pr_alert("Error:Could not initialize /proc/%s\n", PROCFS_NAME);
        return -ENOMEM;
    }

    pr_info("/proc/%s created\n", PROCFS_NAME);
    return 0;
}

static void __exit procfs1_exit(void)
{
#ifdef HAVE_PROC_READ_SUITE
		remove_proc_entry(PROCFS_NAME, NULL);
#else
        proc_remove(our_proc_file);
#endif
    pr_info("/proc/%s removed\n", PROCFS_NAME);
}

module_init(procfs1_init);
module_exit(procfs1_exit);

MODULE_LICENSE("GPL");
```

为了测试方便 Makefile 也随手贴一下，注意路径自己定义

```makefile
KERNEL_DIR=
ARCH=
CROSS_COMPILE=

obj-m += procfs1.o
ccflags-y := -g -DDEBUG

.PHONY: all clean

PWD := $(CURDIR)

all:
        $(MAKE) -C '$(KERNEL_DIR)' M='$(PWD)' ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) V=1 modules

clean:
        $(MAKE) -C '$(KERNEL_DIR)' M='$(PWD)' clean
```

编译后会生成模块文件 procfs1.ko 。在对应的 linux guest 机器上测试

```bash
$ insmod procfs1.ko
$ echo 'HelloWorld!' > /proc/pbuf
procfile write HelloWorld!

$ cat /proc/pbuf
procfile read pbuf
HelloWorld!
copy_to_user failed
copy_to_user failed
```


## 使用说明 {#使用说明}


### 关键接口函数和结构 {#关键接口函数和结构}

从上例可以看出，创建一个 proc 虚拟文件包含了以下的几个步骤

-   定义一个 `struct proc_ops` 或者 `struct file_operations` 的实例，read 和 write
    的函数指向自定义的 `procfile_read` 和 `procfile_write` 回调函数

-   调用函数 `proc_create` 创建一个虚拟文件，并将参数 proc_fops 对应的 `struct
      file_operations` 实例与之关联

-   实现读写的回调函数 `procfile_read` 和 `procfile_write`

-   在模块退出时，调用函数 `remove_proc_entry` 或者 `proc_remove` 将 proc 虚拟文件删除

对应的几个函数都可以在头文件 [include/linux/proc_fs.h](https://github.com/torvalds/linux/blob/v3.9/include/linux/proc_fs.h) 中找到声明

```c
struct proc_dir_entry *proc_create_data(const char *name, umode_t mode,
				struct proc_dir_entry *parent,
				const struct file_operations *proc_fops,
				void *data);
static inline struct proc_dir_entry *proc_create(const char *name, umode_t mode,
	struct proc_dir_entry *parent, const struct file_operations *proc_fops)
{
	return proc_create_data(name, mode, parent, proc_fops, NULL);
}
extern void remove_proc_entry(const char *name, struct proc_dir_entry *parent);
```

主要的数据结构则是 `struct proc_dir_entry`

```c
struct proc_dir_entry {
	// ...
	const struct inode_operations *proc_iops;
    const struct file_operations *proc_dir_ops;

    // ...
};

struct file_operations {
    // ...
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    // ...
};
```

从上面的分析可以知道，最重要的就是 `procfile_read` 和 `procfile_write` 两个回调函数了。write 函数的参数说明如下：

```c
/*

procfile_write函数将数据从用户空间的buff 中拷贝到文件的 (*off) 位置处。

file: [IN] 文件指针

buff: [IN] 需要写入文件的数据起始指针。注意这个buff是在user space的，所以需要使
      用copy_from_user 拷贝到内核空间。同时需要注意 copy_from_user 函数返回的是
      还没有被拷贝到用户空间去的数据长度，而不是像一般的返回拷贝成功的长度。

len: [IN] 需要写入的数据长度

off: [IN/OUT] 表示文件的偏移地址offset。数据应该从 buff 的位置开始写入到文件的
     (*off) 偏移出。每一次被调用后，函数需要根据写入的数据长度调整 (*off) 的值。

return: 成功写入的数据长度
*/
static ssize_t procfile_write(struct file *file,
							  const char __user *buff,
                              size_t len,
							  loff_t *off)

/*
procfile_read 函数负责从文件的（*off）位置拷贝数据到用户空间的 buff 中去。

file: [IN] 文件指针

buff: [OUT] 需要读出的数据的存放位置。注意这个buff是在user space的，所以需要使用
      copy_to_user 拷贝到用户空间。同时需要注意 copy_to_user 函数返回的是还没有
      被拷贝到用户空间去的数据长度，而不是像一般的返回拷贝成功的长度。

len: [IN] 需要读取的数据长度

off: [IN/OUT] 表示文件的偏移地址offset。数据应该从文件的(*off) 偏移处读取到 buff
     中。每一次被调用后，函数需要根据已经读取的数据长度调整 (*off) 的值。

return: 成功读取的数据长度
*/
static ssize_t procfile_read(struct file *file,
							 const char __user *buff,
                             size_t len,
							 loff_t *off)
```

这两个函数的具体说明可以参考 [Linux Device Drivers, Third Edition](https://lwn.net/Kernel/LDD3/) 的第三章讲解的
read 和 write 函数部分，大同小异。


### 深入 procfile_read {#深入-procfile-read}

了解了接口函数的使用，我们可以回到先前的例子再仔细分析一下输出。可以看到在 `cat
/proc/pbuf` 的时候打印了两次的 "copy_to_user failed"。为什么会打印两次呢？

因为 cat 默认是使用 sendfile 函数来直接在 kernel space 中从 `/proc/pbuf` 文件传递数据到 stdout 的。只有当 sendfile 返回值为 0（eof）的时候 cat 才会退出。

对于 `/proc/pbuf` 文件的读回调函数来说，其调用栈如下

```bash
1  procfile_read            procfs1.c    30   0xc87e602a
2  proc_reg_read            inode.c      197  0xc10c82d8
3  do_loop_readv_writev     read_write.c 636  0xc108d7a9
4  do_readv_writev          read_write.c 768  0xc108d9ac
5  vfs_readv                read_write.c 790  0xc108d9eb
6  kernel_readv             splice.c     567  0xc10ab12d
7  default_file_splice_read splice.c     643  0xc10ac9e6
8  do_splice_to             splice.c     1146 0xc10ab496
9  splice_direct_to_actor   splice.c     1217 0xc10ab561
10 do_splice_direct         splice.c     1307 0xc10acce2
11 do_sendfile              read_write.c 968  0xc108ddff
12 sys_sendfile64           read_write.c 1023 0xc108dfde
13 ia32_sysenter_target     entry_32.S   439  0xc12733be
14 ??                                         0xffffe424
15 ??                                         0x804fd47
16 ??                                         0x80e9616
```

由于函数 `splice_direct_to_actor` 中会循环调用 `do_splice_to` 函数来传递数据，直到返回值为 0 （也就是 `/proc/pbuf` 的读回调函数返回 0）。所以其实 `do_sendfile`
被调用了 2 次，而 `procfile_read` 则被调用了 3 次。过程如下所示：

```bash
- do_sendfile (return 13)
  - procfile_read (return 13)
  - procfile_read (return 0)
- do_sendfile (return 0)
  - procfile_read (return 0)
```


### 接口函数的演变 {#接口函数的演变}

为什么在上面的示例代码里那么多的宏？这是因为随着 linux kernel 的不断更新，proc
虚拟文件的使用方式也发生了多次变化。最重要的变化如下表所示（参见头文件
[include/linux/proc_fs.h](https://github.com/torvalds/linux/blob/master/include/linux/proc_fs.h) ）

| VERSION | CHANGE                                                      |
|---------|-------------------------------------------------------------|
| v3.10   | remove read_proc and write_proc from struct proc_dir_entry  |
| v5.6    | use 'struct proc_ops' to instead of 'struct file_operation' |
|         |                                                             |

翻看 3.10 以前的 linux kernel，可以看到 proc 文件系统的核心数据结构 proc_dir_entry
如下所示：

```c
struct proc_dir_entry {
    // ...
	const struct inode_operations *proc_iops; // Inode operations functions
	const struct file_operations *proc_fops;  // File operations functions
    // ...
	read_proc_t *read_proc;                   // proc read function
	write_proc_t *write_proc;                 // proc write function
    // ...
};

struct file_operations {
    // ...
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    // ...
};
```

也就是说读写文件可以通过两种方式进行

-   A. 通过调用 proc 提供的函数指针接口 `read_proc` 和 `write_proc`
-   B. 通过标准的文件系统接口 `proc_fops->read` 和 `proc_fops->write`

再看下 5.6 的数据结构（ [fs/proc/internal.h](https://github.com/torvalds/linux/blob/v5.6/fs/proc/internal.h) ）

```c
struct proc_dir_entry {
	// ...
	const struct inode_operations *proc_iops;
	union {
		const struct proc_ops *proc_ops;
		const struct file_operations *proc_dir_ops;
	};
    // ...
};

struct proc_ops {
	// ...
	ssize_t	(*proc_read)(struct file *, char __user *, size_t, loff_t *);
	ssize_t	(*proc_write)(struct file *, const char __user *, size_t, loff_t *);
    // ...
} __randomize_layout;

```

可以发现， `read_proc` 和 `write_proc` 接口被移除了；同时多了一个 struct
proc_ops 结构指针。之所以把原来的 `struct file_operations` 替换为 `struct
proc_ops` ，原因是 `struct file_operations` 包含了很多 VFS 不需要的成员；替换成
`struct proc_ops` 作为 proc VFS 专用的结构体，可以避免在 `struct
file_operations` 增加新功能的时候影响 proc VFS，也可以通过在 `struct proc_ops`
中添加特定的一些成员来扩展 proc VFS 的功能。

由于 `proc_ops->proc_read` 和 `proc_ops->proc_write` 函数原型和
`file_operations->read` ， `file_operations->write` 完全一样。所以最佳使用方式应该是通过文件系统来读取 proc 的虚拟文件。这样不管是 5.6 之前的 file_operations 或者之后的 proc_ops 都可以使用一样的 read write 函数了。


## 高级应用 {#高级应用}


### 数据量大于 PAGE_SIZE {#数据量大于-page-size}

如果仔细阅读上面的代码就会注意到，这个例子中的 readfile_proc 只能支持数据量小于等于一个 PAGE_SIZE 的情形。如果需要读取大于一个 PAGE_SIZE 的数据，那么函数应该怎么写呢？示例如下：

```c
#define PROCFS_MAX_SIZE		(1024*8)

/* The buffer used to store character for this module */
static char procfs_buffer[PROCFS_MAX_SIZE];
/* The size of the buffer */
static unsigned long procfs_buffer_size = 0;

static ssize_t procfile_read(struct file *file, const char __user *buff,
                             size_t len, loff_t *off)
{
	size_t count = len, data_length = sizeof(procfs_buffer);
    ssize_t retval = 0;
    unsigned long ret = 0;

    if (*off >= data_length)
        goto out;
    if (*off + len > data_length)
        count = data_length - *off;

    /* ret contains the amount of chars wasn't successfully written to `buf` */
    ret = copy_to_user(buff, procfs_buffer + *off, count);
    *off += count - ret;
    retval = count - ret;

out:
    return retval;
}

/*
cat /dev/zero | tr "\0" "a" | dd of=/tmp/1.txt bs=1023 count=1
cat /tmp/1.txt > /proc/pbuf
*/
static ssize_t procfile_write(struct file *file, const char __user *buff,
                              size_t len, loff_t *off)
{
	unsigned long write_size = len;
	unsigned long left_size = PROCFS_MAX_SIZE - procfs_buffer_size;

    if (write_size > left_size)
        write_size = left_size;

	if (copy_from_user(procfs_buffer + procfs_buffer_size, buff, write_size)) {
		pr_info("copy_from_user failed\n");
        return -EFAULT;
	}

	pr_info("procfile write offset %lu with length %lu\n", procfs_buffer_size, write_size);
	procfs_buffer_size += write_size;
	*off += write_size;

    return write_size;
}
```


## proc 专用接口 {#proc-专用接口}

如果是使用的是 3.10 以后的内核版本，那么请出门右转，因为下面的内容对你帮助不大。新的内核已经不支持这种使用方式。但是如果你使用的还是旧版的 kernel，那么请仔细的往下看。因为在网络上搜一下就会发现，很少有文档会去详细解释这个接口怎么使用。或者解释了也是云里雾里，很难看懂。

在看我写的文字之前，建议先学习一下 kernel 自带的文档和例程，毕竟这才是最权威的

-   [Linux Kernel Procfs Guide](http://www.cs.albany.edu/~sdc/CSI500/linux-2.6.31.14/Documentation/DocBook/procfs-guide/index.html)
-   [procfs_example.c](https://github.com/torvalds/linux/blob/v2.6.31/Documentation/DocBook/procfs_example.c)

然后如果还不明白，可以参考我下面的补充说明。


### 实例说明 {#实例说明}

还是老规矩，先举例子，如果你只是想用一下，不想知道太多（很忙），那么照例子写就完了。由于 proc 的专用接口提供了几种写法，所以需要分情况来举例。（就很烦）

由于 write 回调函数相对比较清楚，我只对 read 函数进行分析。


#### 传输数据量小于等于一个 PAGE_SIZE {#传输数据量小于等于一个-page-size}

在早期的 linux kernel 中，有大量的使用 proc 专用接口来读写 proc 虚拟文件的例子。主要有以下的两种

-   写法 A

<!--listend-->

```c
static int ap_debug_proc_read(char *page, char **start, off_t off,
                              int count, int *eof, void *data)
{
    char *p = page;
    struct ap_data *ap = (struct ap_data *) data;

    if (off != 0) {
        *eof = 1;
        return 0;
    }

    p += sprintf(p, "BridgedUnicastFrames=%u\n", ap->bridged_unicast);
    p += sprintf(p, "BridgedMulticastFrames=%u\n", ap->bridged_multicast);

    return (p - page);
}
```

-   写法 B

<!--listend-->

```c
static int ap_debug_proc_read(char *page, char **start, off_t off,
                              int count, int *eof, void *data)
{
    char *p = page;
    struct ap_data *ap = (struct ap_data *) data;

    p += sprintf(p, "BridgedUnicastFrames=%u\n", ap->bridged_unicast);
    p += sprintf(p, "BridgedMulticastFrames=%u\n", ap->bridged_multicast);

	*eof = 1;
    return (p - page);
}
```

如果查看调用这个回调函数的函数 `__proc_file_read` （实现在文件 [fs/proc/generic.c](https://github.com/torvalds/linux/blob/v3.9/fs/proc/generic.c)
中）就可以发现，B 的写法要更好一些，A 则会多循环一次。

这种写法其实也就是在函数 `__proc_file_read` 注释中提到的 3 种做法中的第一种。原文摘录如下:

```bash
Leave *start = NULL.  (This is the default.) Put the data of the requested
offset at that offset within the buffer.  Return the number (n) of bytes there
are from the beginning of the buffer up to the last byte of data.  If the number
of supplied bytes (= n - offset) is greater than zero and you didn't signal eof
and the reader is prepared to take more data you will be called again with the
requested offset advanced by the number of bytes absorbed.  This interface is
useful for files no larger than the buffer.
```

还不理解是不是？换种说法

```bash
1st call:
- *start = NULL
# enter proc_read
- memcpy(page + offset, internal_data + offset, data_copied_to_page_len)
- data_copied_to_user_len = data_copied_to_page_len
- return (offset + data_copied_to_user_len)
# exit proc_read
- copy data of data_copied_to_user_len from kernel space (page + offset) to user
  space (buf + offset)
- offset += data_copied_to_user_len

2nd call:
- *start = NULL
# enter proc_read
- memcpy(page + offset, internal_data + offset, data_copied_to_page_len)
- data_copied_to_user_len = data_copied_to_page_len
- return (offset + data_copied_to_user_len)
# exit proc_read
- copy data of data_copied_to_user_len from kernel space (page + offset) to user
  space (buf + offset)
- offset += data_copied_to_user_len
```

在这种情况下，start 不被使用。需要关注的只有 offset 和 eof 。

从前面的函数调用可以看出，虽然一般用 eof 来判断是否读取结束，但是即使没有设置
eof 也能结束读取循环。只要把数据读取结束即可。只是这样容易出错且效率低，所以显式的指定 eof 是个更好的编程习惯。

如果对于 offset 和 eof 的理解还是不够清楚，可以参考 stackoverflow 上的一段举例，我觉得还是挺清楚的。

```bash
off is the position in the file from where data has to be read from. This is
like off set of normal file. But, in the case of proc_read it is some what
different. For example if you invoke a read call on the proc file to read 100
bytes of data, off and count in the proc_read will be like this:

in the first time, off = 0, count 100. Say for example in your proc_read you
have returned only 10 bytes. Then the control cannot come back to the user
application, your proc_read will be called by the kernel once again with off as
10 and count as 90. Again if you return 20 in the proc_read, you will again
called with off 30, count 70. Like this you will be called till count reaches
0. Then the data is written into the given user buffer and your application
read() call returns.

But if you don't have hundred bytes of data and want to return only a few bytes,
you must set the eof to 1. Then the read() function returns immediately.
```


#### 传输数据量大于一个 PAGE_SIZE {#传输数据量大于一个-page-size}

还是先看例子

```c
#define BUFSIZ 8192
static char output_data[BUFSIZ] = {0};

static int read_method2(char *page,
						char **start,
						off_t offset,
						int count,
						int *eof,
						void *data)
{
    int len = strlen(output_data);
	int read_size = PAGE_SIZE-1; // suppose PAGE_SIZE is 4K

    if (offset >= len) {
        *eof = 1;
		return 0;
	}
    memcpy(page, output_data+offset, read_size);
	*start = read_size;

    return (read_size);
}
```

所以如果 caller 请求读取的是 8K 字节，那么第一次调用 read_proc 的时候 offset 0，
count 8192；第二次 `offset = offset + (*start)` ，即 4095 (4K-1)，count 4097；第三次 offset 8190，count 2 ...

这里使用的是 3 种方式中的第二种做法。原文如下

```bash
1) Set *start = an unsigned long value less than the buffer address but greater
than zero. Put the data of the requested offset at the beginning of the buffer.
Return the number of bytes of data placed there.  If this number is greater than
zero and you didn't signal eof and the reader is prepared to take more data you
will be called again with the requested offset advanced by *start. This
interface is useful when you have a large file consisting of a series of blocks
which you want to count and return as wholes.
```

这个比较难理解一些。因为首先 `*start` 需要设置成一个 unsigned long 的值，并且要大于 0 小于 page 的地址（比如 0x80002000）。在这种情况下，offset 会根据 `*start`
的值进行递增。

首先需要注意第一次调用 read_proc 的时候 `*start` 必定是 NULL。这是函数
`__proc_file_read` 里面写死的。具体一些，上面这段注释可以按以下逻辑理解：

```bash
1st call:
- *start = NULL
# enter proc_read
- memcpy(page, internal_data + offset, data_copied_to_page_len)
- *start = N (0 < *start < page)
- data_copied_to_user_len = data_copied_to_page_len
- return data_copied_to_user_len
# exit proc_read
- copy data of data_copied_to_user_len from kernel space(page) to user space
- offset += *start

2nd call:
- *start >= page
# enter proc_read
- memcpy(page, internal_data + offset, data_copied_to_page_len)
- *start = N (0 < *start < page)
- data_copied_to_user_len = data_copied_to_page_len
- return data_copied_to_user_len
# exit proc_read
- copy data of data_copied_to_user_len from kernel space(page) to user space
- offset += *start
```

这里其实有个隐藏的条件。由于每次 offset 是根据 `*start` 递增，而用户空间的 buf 却是根据返回值（n）递增，所以 n 应该和 `*start` 相等。又 buf 是从 page 中的数据拷贝而来，所以拷贝的数据长度（data_copied_to_user_len，data_copied_to_page_len）应该和 \*start 相等才合理。

这里有个问题需要注意。由于每次都是按照 `*start` 的长度来拷贝 page 中的内容到
user space 去的，那么如果最后一次 read_proc 被调用的时候 `*start` 被设置成了一个超出剩余长度的值，那么就会从 page 中拷贝多余的信息到用户空间。另外，由于
`__proc_file_read` 中的控制循环的变量 nbytes 是一个无符号型的数，所以永远不会 `<
0` ，只有在 `= 0` 的时候才可能退出。因此在这种情况下，如果没有用 \*eof 显式的指定结束而退出，那么有可能就进入一个死循环了。所以 `*start` 的值应该根据当前剩余数据的长度进行调整。

这种方式和第一种最大的区别在于返回值。当 `start == NULL` 的时候返回值应该是 page
中所有数据的长度（offset + 拷贝的长度），而当 `start > 0 && start < (unsigned
long)page` 的时候，返回值只是拷贝的数据长度。

另外一种方式就是函数 `__proc_file_read` 注释中提到的最后一种了。这种方式也可以处理数据量大于一个 PAGE_SIZE 的情形。

示例代码如下：

```c
static int hp_sdc_rtc_read_proc(char *page, char **start, off_t off,
								int count, int *eof, void *data)
{
    int len = hp_sdc_rtc_proc_output (page);
    if (len <= off+count) *eof = 1;
    *start = page + off;
    len -= off;
    if (len>count) len = count;
    if (len<0) len = 0;
    return len;
}
```

原文摘录如下：

```bash
2) Set *start = an address within the buffer. Put the data of the requested
offset at *start. Return the number of bytes of data placed there. If this
number is greater than zero and you didn't signal eof and the reader is prepared
to take more data you will be called again with the requested offset advanced by
the number of bytes absorbed.
```

首先需要注意第一次调用 read_proc 的时候 start 必定是 NULL。这是函数~\__proc_file_read~ 里面写死的。所以上面这段注释所对应的行为必然是从第二次进入才会发生。我们可以这么理解

```bash
1st call:
- *start = NULL
# enter proc_read
- memcpy(page, internal_data, data_copied_to_page_len)
- *start = page + offset (*start >= page)
- data_copied_to_user_len = data_copied_to_page_len - offset
- return data_copied_to_user_len
# exit proc_read
- copy data of data_copied_to_user_len from kernel space(*start) to user space
- offset += data_copied_to_user_len
# 注意read_proc 调用者的目的是从文件的offset处读取，所以当把数据放在page中时，需
# 要再加上offset赋值给 *start

2nd call:
- *start >= page
# enter proc_read
- memcpy(page, internal_data, data_copied_to_page_len)
- *start = page + offset (*start >= page)
- data_copied_to_user_len = data_copied_to_page_len - offset
- return data_copied_to_user_len
# exit proc_read
- copy data of data_copied_to_user_len from kernel space(*start) to user space
```


## 参考 {#参考}

-   [The Linux Kernel Module Programming Guide](https://sysprog21.github.io/lkmpg/)
-   [Access the Linux kernel using the /proc filesystem](https://developer.ibm.com/articles/l-proc/)
-   [Linux Kernel Procfs Guide](http://www.cs.albany.edu/~sdc/CSI500/linux-2.6.31.14/Documentation/DocBook/procfs-guide/index.html)
-   [What is the difference between procfs and sysfs?](https://unix.stackexchange.com/questions/4884/what-is-the-difference-between-procfs-and-sysfs)

