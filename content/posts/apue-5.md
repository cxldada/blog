---
title: 5.进程环境
date: 2020-07-21 10:14:43
draft: false
categories: 
- APUE
---

在学习系统进程`API`之前，我们要先熟悉Unix系统的进程环境。这里我们要知道存储空间布局是什么样式，进程如何使用环境变量，进程的各种终止方式，还有`longjmp`和`setjmp`函数以及它们与栈的交互作用，还要知道如何查看进程的资源限制

<!--more-->

# main函数

当内核执行C程序时，在调用main函数之前会先调用一个特殊的启动例程。

可执行程序文件将此启动例程指定为程序的起始地址----这是由连接编辑器设置的，连接编辑器是由C编译器调用的。

启动例程从内核获取命令行参数和环境变量值，然后为按上述方式调用main函数做好准备

# 进程终止

有8种方式终止进程，其中有5种是正常终止：

1. 从`main`函数返回
2. 调用`exit`函数
3. 调用`_exit`或`_Exit`
4. 最后一个线程从启动例程返回
5. 最后一个线程调用`pthread_exit`

还有3种异常终止：

1. 调用`abort`
2. 接到一个信号
3. 最后一个线程对取消请求作出响应

## 停止进程的函数

下面有3个函数用来正常终止一个程序

```c
#include <stdlib.h> /* 这两个函数由IOS C说明 */

void exit(int status);
void _Exit(int status);

#include <unistd.h> /*这个函数由POSIX说明 */
void _exit(int statu);
```

`_exit`和`_Exit`立即进入内核，而`exit`则先执行一些清理工作，然后返回内核

三个函数都有一个`status`参数，称为终止状态。如果：

* 调用这些函数时不带终止状态
* `main`函数内执行了一个无返回值的`return`语句，
* `main`函数没有声明返回类型为整型

则该进程的终止状态是未定义的。

如果`main`的返回类型是整型，且`main`执行到最后一条语句时返回，那么该终止状态为0

`main`函数返回一个整数值与用该值调用`exit`是等价的

## 终止处理函数

这个函数用来注册退出前要调用的函数

```c
#include <stdlib.h>

int atexit(void (*func)(void));
// 成功返回0，出错返回非0
```

使用`atexit`注册的函数和在`exit`中调用的顺序是相反的，类似于栈先进后出

同一个函数注册多次，也会调用多次

如果进程调用`exec`函数族中的任一函数，则会清除所有已注册的终止处理程序

# 命令行参数

`ISO C`和`POSIX`都要求`argv(argc)`是一个空指针。这就使得我们可以将参数处理循环改为如下格式

```c
for (int i = 0; argv[i] != NULL; ++i)
```

# 环境表

每个进程都会有一张环境表。与参数表一样，环境表也是一个字符指针数组。

全局变量`environ`包含了该指针数组的地址：`extern char **environ;`

里面的每一项的格式：`name = value`

# C程序的存储空间布局

C程序由下列几部分组成：

* 正文段：存放由CPU执行的机器指令部分。
  * 正文段是可共享的，即使频繁的执行程序，存储器中也只需一个副本即可
  * 通常这部分是只读的，以防止程序由意外而修改其指令
* 初始化数据段。在**任何函数之外**，需要**明确赋初值**的变量，都存放在初始化数据段中。
* 未初始化数据段。在**任何函数之外声明的**函数，内核会将此段中的数据初始化为0或空指针。
* 栈。自动变量以及每次函数调用时所需保存的信息都存放在此段中。
* 堆。通常在堆中进行动态存储分配。由于历史惯例，堆位于未初始化数据段和栈之间。

上面列出的几部分是典型的存储空间布局，但不一定是所有系统都是这么实现的。下面典型的存储空间安排图

![](https://raw.githubusercontent.com/cxldada/imgs/master/202403121148978.png)

未初始化数据段不会存放在数据磁盘程序文件中，因为它是由`exec`设置的。只有正文段和初始化数据段是放在磁盘程序文件中的

**可以使用`size`命令查看程序的正文段、数据段和`bss`段的长度**

# 共享库

共享库使得可执行文件中不再需要包含共用的库函数，而只需要在所有进程都可以引用的存储区中保存这种库例程的一个副本。

程序一次执行或者第一次调用某个库函数时，用动态链接方法将程序与共享库函数相连接。

共享库的好处是减少了每个可执行文件的长度，但增加了一些运行时间开销，这种时间开销发生在第一次调用的时候。

使用共享库还有一个优点，就是可以用库函数的新版本替换老版本而不用重新链接编辑，当然前提是库函数的名字和参数没有发生变化

# 存储空间的分配

`ISO C`说明了3种用于存储空间动态分配的函数：

```c
#include <stdlib.h>

void *malloc(size_t size);
void *calloc(size_t nobj, size_t size);
void *realloc(void *ptr, size_t newsize);
// 成功返回非空指针，出错返回NULL

void free(void *ptr);
```

三个动态分配空间函数的作用：

1. `malloc`，分配指定字节数的存储区。新分配的存储区中的初始值不确定
2. `calloc`，为指定数量指定长度的对象分配存储空间。新分配的存储区中的每一位都被初始化为0
3. `realloc`，增加或减少以前分配区的长度。如果是增加的话，那么新增部分的初始值不确定

## 细节说明

* 这三个函数返回的指针都是适当对其的，使其可用于任何数据对象。

* `free`函数释放`ptr`所指向的存储空间。被释放的空间通常被送入可用存储区池，以后，可在调用上面三个分配函数的时候再分配
* 如果`realloc`函数中的`ptr`是空指针，那么作用和`malloc`函数的作用一样
* 这些函数通常会使用`sbrk`系统调用来实现。
* 大多数实现所分配的存储空间都比所要求的的稍大一些，额外的空间用来记录管理信息——分配块的长度、指定下一分配块的指针等等信息
* 有一些比较致命的错误：
  * 释放一个已经释放了的块
  * 调用`free`时所用的指针不是3个`alloc`函数的返回
  * 忘记`free`分配的块，会导致内存泄漏

# 环境变量

环境字符串的形式如下：

`name=value`

UNIX系统并不查看这些字符串，它们的解释完全取决于各个引用程序。

## 获取环境变量值的函数

```c
#include <stdlib.h>
char *getenv(const char *name);
```

* 此函数返回一个指针，它指向`name=value`中的`value`。
* 我因该使用这个函数来获取环境变量值，而不是使用环境表指针`environ`

## 设置环境变量的函数

除了获取环境变量的值，我们还可以设置环境变量的值，当然后面的学习中会讲到，设置当前进程的环境变量值，并不会影响父进程的环境，但是会影响它本身和它其后产生的进程。函数声明如下：

```c
#include <stdlib.h>
int putenv(char *str);
// 成功返回0，出错返回非0

int setenv(const char *name, const char *value, int rewrite);
int unsetenv(const char *name);
// 成功返回0，出错返回-1
```

* `putenv`函数将形如`name=value`的字符串，放到环境变量中，如果`name`已经存在，则先删除原来的值
* `setenv`函数，将`name`和`value`作为函数参数进行传递。如果`name`已经存在，则
  * 若`rewrite`非0，则先删除
  * 若`rewrite`为0，则不删除，也不设置，并且不会出错返回
* `unsetenv`删除`name`的定义，即使不存在也不会出错。

注意：`setenv`必须要分配存储空间，而`putenv`不需要。所以有很多软件都是用`putenv`，但是还是建议分配存储空间，不然临时变量的空间被重用是会导致设置的变量错误

## 分析修改环境变量的函数

> 修改环境变量的值是比较复杂的，查看上面存储空间分配一节中的表格图，可以知道，环境变量和命令行参数是在空间分布图的上方，也就是高地址。
>
> 那么删除一个字符串很简单，只需要将字符串后面的指针，陆续向前移动一个就可以了。
>
> 但是增加一个就很麻烦了。因为环境表通常占用进程地址空间的顶部，所有它不能再向高地址方向扩展。同时也不能移动它之下的各栈帧，所以它也不能向低地址扩展。
>
> 下面就分析一下增加环境变量时，这些函数是如何实现的

1. 如果修改一个现有的`name`
   1. 如果新`value`的长度小于或等于现有`value`长度，则只要将新字符串复制到原字符串所用的空间中
   2. 如果行`value`的长度大于原长度，则必须调用`malloc`为新字符串分配空间，然后将新字符串复制到该空间中，接着使环境表中针对`name`的指针指向新分配区。
2. 如果新增加一个`name`就复杂了
   1. 如果这是第一次增加新`name`,则必须调用`malloc`为新的**指针表**分配空间。接着，将原来的环境表复制到新分配区，并将指向新`name=value`字符串的指针存放在该指针表的尾部，然后又将一个空指针存放在其后，作为结束符。最后使environ指向新指针表。这样的话，想象一下空间分配图，环境表指针指向了堆中的一块存储区，然后改存储区的每项又指向顶部的环境变量值
   2. 如果不是第一次增加`name`。则可知以前调用`malloc`在堆中为环境表分配了空间，所有只需要调用`realloc`函数，修改存储区的大小，在将新加的指针存入尾部，然后在后面跟一个空指针就可以了

# 非局部跳转

C语言中，goto可以实现在函数内部跳转功能，但是它不能够跨函数执行，所以就有了下面两个函数，用来进行跨函数跳转

```c
#include <setjmp.h>

int setjmp(jmp_buf env);
// 直接调用返回0，若从longjmp函数返回，则返回非0
void longjmp(jmp_buf env,int val);
```

* `jmp_buf`参数是用来存储调用`longjmp`时，能用来恢复栈状态的所有信息
* `setjmp`和`longjmp`中的的`jmp_buf`参数应该是同一个，所以最好定义为全局变量
* `longjmp`的`val`参数是`setjmp`的返回值。这样的话就可以在有多个`longjmp`函数的时候，区分到底调用的是哪一个`longjmp`函数

调用这两个函数的时候有一个问题：自动变量、寄存器变量和易失变量，他们是恢复到`setjmp`时的值，还是保持`longjmp`时的值。这个答案是不确定的，需要查看你所使用的UNIX系统的手册

# 查询资源限制的函数

每个进程都有一组资源限制，其中一些可以用下面两个函数查询和更改

```c
#include <sys/resource.h>
int getrlimit(int resource,struct rlimit *rlptr);
int setrlimit(int resource,const struct rlimit *rlptr);
// 成功返回0，出错返回非0

struct rlimit
{
    rlim_t rlim_cur;	// 软限制：当前限制
    rlim_t rlim_max;	// 硬限制：软限制可以设置的最大值
};
```

更改资源限制时，必须遵循下列3条规则：

1. 任何一个进程都可以将一个软限制更改为小于或等于其硬限制值
2. 任何一个进程都可以降低其硬限制，但它必须大于或等于其软限制。**这种限制普通用户而言是不可逆的**
3. 只有root用户可以提高硬限制值

常量`RLIM_INFINITY`指定了一个无限量的限制

`resource`参数的可选值：

|        字段         |                             说明                             |
| :-----------------: | :----------------------------------------------------------: |
|     `RLIMIT_AS`     |            进程总的可用存储空间的最大长度（字节）            |
|    `RLIMIT_CORE`    |           core文件的最大字节数，0阻止创建core文件            |
|    `RLIMIT_CPU`     |  CPU时间的最大值(秒)，超过此软限制会向进程发送`SIGXCPU`信号  |
|    `RLIMIT_DATA`    |                      数据段的最大字长度                      |
|   `RLIMIT_FSIZE`    |                 可以创建的文件的最大字节长度                 |
|  `RLIMIT_MEMLOCK`   |     一个进程使用`mlock`能够锁定在存储空间的最大字节长度      |
|  `RLIMIT_MSGQUEUE`  |          进程为`POSIX`消息队列可分配的最大存储字节           |
|    `RLIMIT_NICE`    |        为了影响进程的调度优先级，nice值可设置的最大限        |
|   `RLIMIT_NOFILE`   | 每个进程能打开的最多文件，修改此值会影响`sysconf`函数在参数`_SC_OPEN_MAX`中的返回值 |
|   `RLIMIT_NPROC`    | 每个实际用户ID可拥有的最大子进程数，修改此值会影响`sysconf`函数在参数`_SC_CHILD_MAX`中的返回值 |
|    `RLIMIT_NPTS`    |               用户可同时打开的伪终端的最大数量               |
|    `RLIMIT_RSS`     |                     最大驻内存集字节长度                     |
|   `RLIMIT_SBSIZE`   | 在任一给定时刻，一个用户可以占用的套接字缓冲区的最大长度（字节） |
| `RLIMIT_SIGPENDING` |                 一个进程可排队的信号最大数量                 |
|   `RLIMIT_STACK`    |                       栈的最大字节长度                       |
|    `RLIMIT_SWAP`    |               用户可消耗的交换空间的最大字节数               |
|    `RLIMIT_VMEM`    |                      与`RLIMIT_AS`同义                       |

* 这些限制值的修改会影响该进程本身和其产生的子进程。
