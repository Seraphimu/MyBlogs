# 文件I/O基础

> 一提到文件大家第一时间想到的可能就不在磁盘上躺着的游戏、音乐、视频。的确如此，可如果这些文件只存放在磁盘上，它们能被“操作（增删查改）”吗？根据“存储程序控制原理”，磁盘文件必须载入内存才能被处理。对文件操作的前提是进程打开文件，然后才能对其进行操作。

### 'open()' 系统调用

函数原型：

```c
int open(const char *pathname, int flags, .../* mode_t mode */ );
```

作用: 打开一个文件, 返回**文件描述符**.

参数:

- `pathname`: 要打开文件的路径
- `flags`: 打开文件的方式
- `mode`: 创建文件的权限

返回值: 

- 成功返回文件fd
- 失败返回 -1

文件描述符就是进程描述文件的方式, 或者说是进程找到文件的方式, 本质上是一个无符号整数.

<!-- 内核数据结构应该放在哪里谈呢? -->

打开文件的方式常用的有如下几种:

- O_RDONLY: 只读方式打开
- O_WRONLY: 只情怀方式打开
- O_CREAT:如果文件不存在, 创建之
- O_TRUNC: 覆盖原有内容
- O_WRONLY:读写方式打开


实例: 打开一个文件, 并打印其文件描述符 `fd`

```c
#include <stdio.h>
#include <assert.h>
#include <fcntl.h>
#include <sys/stat.h>

int main(void) {
    umask(0);
    int fd = open("log.txt", O_WRONLY | O_CREAT, 0666);
    assert(fd >= 0);

    printf("fd = %d\n", fd);

    return 0;
}
```

嗯, 编译运行之后的结果是: 3? 怎么会是3而不是0, 1, 2呢?

```bash
vivit@Xen:test_file$ ./a.out 
fd = 3
```

还记得学C的时候有这么几个老朋友吗? 

- 标准输入: stdin
- 标准输出: stdout
- 标准错误: stderr

笔者在学习系统编程之前对这些东西的印象也十分甚至是九分模糊了. 只能记得它们分别对应 0, 1, 2. 但根本不知道这意味着什么. 但在学习了文件I/O之后就能够明白了.

首先, 每个进程都有自己的 `task_struct` 等数据结构. 因为操作系统不止一个进程, 所以这些进程需要被管理起来, 操作系统的设计者需要把它们描述成一个一个一个的结构体, 再通过合适的数据结构组织起来. 对于进程如此, 对于文件亦是如此.

系统中一定不止一个被打开的文件, 所以这些文件要不要被管理起来? 毫无疑问: 是的! 那么怎么管理呢? 思路同进程, 先把这些文件抽象成一个一个对象, 再用特定的数据结构把它们组织起来. 说到这里, 我们来翻一翻内核源码(基于linux2.6.1):

在 `tash_struct` 下有这么一个字段: `struct files_struct *files;` 这个结构体是负责管理被进程打开的文件的. 继续向下翻, 找到 `struct files_struct` 结构:

```c
struct files_struct {
        atomic_t count;
        spinlock_t file_lock;     /* Protects all the below members.  Nests inside tsk->alloc_lock */
        int max_fds;
        int max_fdset;
        int next_fd;
        struct file ** fd;      /* current fd array */
        fd_set *close_on_exec;
        fd_set *open_fds;
        fd_set close_on_exec_init;
        fd_set open_fds_init;
        struct file * fd_array[NR_OPEN_DEFAULT];
};
```

重点在最后一个字段, 这是一个指针数组, 一个指向 `struct file *` 类型的指针数组. 继续向下翻:

```c
struct file {
	struct list_head	f_list;
	struct dentry		*f_dentry;
	struct vfsmount         *f_vfsmnt;
	struct file_operations	*f_op;
	atomic_t		f_count;
	unsigned int 		f_flags;
	mode_t			f_mode;
	loff_t			f_pos;
	struct fown_struct	f_owner;
	unsigned int		f_uid, f_gid;
    //...
};
```

这就是操作系统用来描述文件的结构体了. 每个被打开的文件都会被操作系统描述成这样一个结构体. 其中 `f_count` 字段的作用是 "引用计数" 意思是这个结构在内存中只需要存在一份, 如果有其他进程也打开该文件, 则无需再创建一个新的 `struct file` 对象, 直接在引用计数 +1 即可. 而关闭文件也并不意味着真正的关闭， 而是引用计数 -1, 直到引用计数变成0, 该文件才会被真正关闭. 这里和硬链接的逻辑有些像.

到这一步, 我们就可以解释为什么打开的新文件它的文件描述符 `fd` 为3了. 对于一个C/C++程序, 默认打开的3个文件: stdin, stdout, stderr被描述成了一个一个 `struct file` 类型的对象, 并且它们的地址被存放在 `struct file * fd_array[]` 中, 所以0, 1, 2的意思是, stdin, stdout, stderr分别占用了该指针数组的0, 1, 2位的下标. 由此, 进程就可以通过 `task_struct` 对自己打开的文件进行操作了.

> 所以文件描述符fd的本质, 就是一个数组的下标, 这个数组记录着该进程打开的所有文件的地址!

而此时, 我们打开一个新的文件, 其`struct file`对象的指针自然就会被放入最小未使用的3号下标的位置了. 那么,如何证明文件fd的分配规则是最小未使用下标呢? 这里介绍一个系统调用: `close()` 系统调用. 它的原型是:

```c
int close(int fd);
```

作用: 根据一个文件fd关闭文件. 

参数:

- fd: 打开文件的文件fd

返回值: 

- 成功: 返回0
- 失败: 返回-1

**实例:** 从键盘输入0, 1, 2任意一个数字, 关闭其文件fd, 再打开一个新的文件, 并打印其被分配的文件fd.

```c
#include <stdio.h>
#include <assert.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>

int main(int argc, char * argv[]) {

    if (argc == 2) {
        //atoi()把字符串转成整数
        int closeFd = atoi(argv[1]);
        if (closeFd == 0 || closeFd == 1 || closeFd == 2) {
            //关闭0, 1, 2任意一个fd
            close(closeFd);

            //此时打开文件，并打印fd
            int fd = open("log.txt", O_WRONLY);
            assert(fd >= 0);
            
            printf("新打开文件的fd: %d\n", fd);

        }
    }
    else {
        printf("./a.out [未使用open打开任意文件前要关闭的文件fd(0, 1, 2)]\n");
    }

    return 0;
}
```

我们可以看到, 输入0,2的时候结果确实如我们所料: 操作系统为新打开的文件分配了**最小未使用**的文件fd. 但很奇怪啊, 为什么 `close(1)` 的时候没有输出呢?

```bash
vivit@Xen:test_file$ ./a.out 0
新打开文件的fd: 0
vivit@Xen:test_file$ ./a.out 1
vivit@Xen:test_file$ ./a.out 2
新打开文件的fd: 2
vivit@Xen:test_file$ 
```

真的没有输出吗? 请看!

```bash
vivit@Xen:test_file$ cat log.txt 
新打开文件的fd: 1
vivit@Xen:test_file$ 
```

为什么原本要被打印在屏蔽上的文字却跑到了文件里? 还记得我们说过, 在C/C++程序中标准输出文件对应的是数组下标1吗? 现在它被换成了我们自己打开的文件, 所以C/C++把原本要打印在屏蔽文件上的字符串"打印"到了文件 `log.txt` 中. 这就叫 **输出重定向**, 用另一个文件接管标准输出.

<!-- 妈的, 写到时这里我是越来越慌了啊, 写着写着才发现原来标准输入/输出是C/C++规定的特性, 还有, 我C/C++是无脑向fd为0的文件读吗?无脑向fd为1的文件中写吗?越写越觉得自己无知!先发! 这就是用输出倒逼输入啊! -->