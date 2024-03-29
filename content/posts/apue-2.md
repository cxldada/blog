---
title: 2. 文件和目录
date: 2020-07-15 08:13:25
draft: false
categories: 
- APUE
---

前面学习了一些文件的I/O操作，对文件的读写等操作都有一定的了解了。
现在通过学习`stat`结构中的字段，详细了解一下文件和目录的属性以及对这些属性操作的函数。


<!--more-->

## stat函数
获取文件信息

### 函数定义

```c++
#include <sys/stat.h>

int stat(const char *restrict pathname,struct stat *restrict buf);
int fstat(int fd,struct stat *buf);
int lstat(const char *restrict pathname,struct stat *restrict buf);
int fstatat(int fd,const char *restrict pathname,struct stat *restrict buf,int flag);
// 这四个函数的返回值：若成功都返回0，出错则返回-1
```

### 参数说明

* `fd`：文件描述符
* `pathname`：文件路径名
* `buf`结构体
```c
struct stat {
    mode_t 		st_mode;  // 文件类型和文件权限
    ino_t 		st_ino;	// iNode节点
    dev_t 		st_dev;	// 设备号码
    dev_t 		st_rdev;	// 特殊设备号码
    nlink_t 	st_nlink;	// 链接数
    uid_t 		st_uid;	// 所属的用户ID
    git_t 		st_gid;	// 所属的组ID
    off_t 		st_size;	// 文件的字节大小
    struct timespec st_atime;	// 最后访问的时间 a = access
    struct timespec st_mtime;	// 最后修改的时间 m = modification
    struct timespec st_ctime;	// 文件状态最后一次修改的时间 c = change
    blksize_t st_blksize;	// 最佳的I/O块大小
    blkcnt_t st_blocks;		// 分配的磁盘块数量
};

struct timespec {
    time_t tv_sec;  // 秒
    long tv_nsec;	// 纳秒
};
```

### 细节说明

* `stat`和`fstat`函数返回与参数一说明的文件相关的信息结构
* `lstat`函数与`stat`相同，但是当文件是一个符号链接时，lstat返回符号链接本身的有关信息
* `fstatat`函数的`flag`参数与`openat`相同，可以参考
	* `flag`为`AT_SYMLINK_NOFOLLOW`时，获取的信息不会跟随符号链接，而是符号链接本身的

## 文件类型

> 文件类型由 stat.st_mode指定
>
> 本节所使用的判断函数都定义在 `<sys/stat.h>`文件中

* 普通文件：
	* 是最常用的文件类型。数据可以是文本或二进制，对于这些数据的解释，取决于处理它们的程序。
	* 使用 **S_ISREG(stat.st_mode)** 进行判断
* 目录文件：
	* 这种文件包含了其他文件的名字以及指向与这些文件有关信息的指针。
	* 对于一个目录文件具有读权限的任一进程都可以读目录的内容，但是只有内核可以直接写目录文件
	* 使用 **S_ISDIR(stat.st_mode)** 进行判断
* 块特殊文件：
	* 这种类型的文件提供对设备**带缓冲**的访问，每次访问以**固定长度**为单位进行。
	* 使用 **S_ISBLK(stat.st_mode)** 进行判断
* 字符特殊文件
	* 这种类型的文件提供对设备**不带缓冲**的访问，每次访问**长度可变**。
	* 系统中的**所有设备**要么是字符特殊文件，要么是块特殊文件
	* 使用 **S_ISCHR(stat.st_mode)** 进行判断
* FIFO
	* 这种类型的文件用于进程间通信
	* 使用 **S_ISFIFO(stat.st_mode)** 进行判断
* 套接字
	* 这种类型的文件用于进程间的网络通信。
	* 套接字也可以用于在一台宿主机上进程之间的非网络通信
    * 使用 **S_ISSOCK(stat.st_mode)** 进行判断
* 符号链接
	* 这种类型的文件指向另一个文件
	* 使用 **S_ISLNK(stat.st_mode)** 进行判断

> POSIX允许实现将进程间通信（IPC）对象说明为文件，并使用以下函数解释对象类型：
>
> 1. S_TYPEISMQ(stat)：消息队列
> 2. S_TYPEISSEM(stat)：信号量
> 3. S_TYPEISSHM(stat)：共享存储对象

----

## 设置用户ID和设置组ID
1. 与一个**进程**相关的ID有6个或者更多：
   * **实际用户ID**和实际组ID：标识了当前运行该进程的究竟是谁
   * **有效用户ID**和**有效组ID**以及**附属组ID**：决定了我们的文件访问权限
   * **保存的设置用户ID**和**保存的设置组ID**(由exec函数保存)：这两个ID是执行一个程序时包含的有效用户ID和有效组ID的副本

通常，有效用户ID和有效组ID，都对应等于实际用户ID和实际组ID

2. 与**文件**相关的ID：
   * 每个文件都有一个所有者ID由`stat`中的`st_uid`指定，和一个组所有者ID由`stat`中的`st_gid`指定的
   * 设置用户ID位：当执行一个程序文件的时候，将进程的**有效用户ID**设置成文件的**所有者ID**
   * 设置组ID位：当执行一个程序文件时候，将进程的**有效组ID**设置成文件的**所属组ID**

比如，当一个程序文件的所有者是root，并且设置了**设置用户ID位**，那么，当执行此进程文件时，这个进程就是一个root进程。(最常见的例子就是`passwd`命令，它会修改`/etc/passwd`文件)
设置用户ID和设置组ID都包含在文件的`st_mode`值中。这两位分别用**常量**`S_ISUID`和`S_ISGID`测试，在linux中这两个是值不是函数。

----

## 文件访问权限

每个文件有9个访问权限位，分为3类：
* 用户读、写和执行
* 组读、写和执行
* 其他读、写和执行

这9个访问权限位由不同的函数以各种方式使用，下面是一些使用方式的简要总结：
1. 当我们用名字打开任何一种类型的文件时，对该名字中包含的每一个目录，我们都必须要有**执行权限**。比如说我要打开`/home/cxl/aa.txt`文件，那么我必须对 `/`、`/home`、`/home/cxl` 这个几个目录有执行权才可以
   1. 目录的读权限和执行权限是不一样的。读权限可以让我读取目录中记录的数据，而执行权限是让我们通过该目录找到一个特定的文件
2. 一个文件的读权限决定了我们能否对文件进行读取内容的操作，这与`open`函数的`O_RDONLY`和`O_RDWR`标志相关
3. 写权限决定了我们能否对文件进行写入内容的操作，这与`open`函数的`O_WRONLY`和`O_RDWR`标志相关
4. 为了在open函数中使用`O_TRUNC`标志，必须对文件具有**写权限**
5. 为了在目录中创建一个新文件，必须对目录具有**写权限和执行权限**
6. 为了删除目录中的文件，必须对包含该文件的目录具有**写和执行权限**，对文件本身没有要求
7. 使用exec系列函数中的任何一个去执行某个文件，必须对文件具有执行权限，而且还**必须是一个普通文件**

一个进程在对文件进行操作时内核会用以下测试方式：
1. 若进程的有效用户ID是0(root)，则允许访问
2. 如果进程的**有效用户ID**等于文件的**所有者ID**，那么操作权限就看文件的用户文件权限位了
3. 如果进程的**有效组ID**或者进程的**附属组ID**之一等于文件的**组ID**，那么操作权限就看文件的组文件权限位了
4. 如果其他用户适当的访问权限被设置，则允许访问，否则拒绝访问

----

## 新文件和目录的所有权

* 新文件的**所有者**被设置为进程的**有效用户ID**
* 新文件的所属组则有两种情况：
	* 新文件的**所属组**被设置为进程的**有效组ID**
	* 新文件的**所属组**被设置为**所在目录的组ID**

组ID的是设定需要看具体的实现，FreeBSD和macOS总是使用第二种实现。
而linux则是可选的，只有在包含该文件的目录的**设置组ID位**被设置的情况下，linux才会使用第二种情况

----

## access和faccess函数

使用进程的**实际用户ID**和**实际组ID**测试对文件的访问权限

### 函数定义

```c
#include <unistd.h>

int access(const char *pathname, int mode);
int faccess(int fd, const char *pathname, int mode, int flag);
// 成功返回0，出错返回-1
```

### 参数说明

* `mode`参数：
	* `F_OK`：文件存在
	* `R_OK`：可读
	* `W_OK`：可写
	* `X_OK`：可执行

### 细节说明

faccessat函数在两种情况下和access函数是相同的：
* pathname是绝对路径
* fd等于AT_FDCWD，并且pathname是相对路径

如果不是上面两种情况的话，faccessat函数计算相对于打开目录(fd)的pathname
flag参数是可以改变faccessat函数的行为的
* 如果flag等于AT_EACCESS的话，访问检查用的是进程的有效用户ID和有效组ID

## umask函数

设置文件模式创建屏蔽字。
文件模式创建屏蔽字就是说在创建一个文件的时候，会给文件设置一个权限，用这个权限值再减去我们使用umask函数设置的值，就是文件的最终权限，所以我们把umask设置的这个值就叫做文件模式创建屏蔽字，因为是要屏蔽的嘛。

### 函数定义

```c
#include <sys/stat.h> mode_t umask(mode_t cmask);
// 返回之前的值
```

### 参数说明

* `cmask`参数：就是那9个文件权限按位或得到的值

### 细节说明

* `umask`值通常在登录时，由`shell`的启动文件设置一次，然后不再改变
* 更改进程的文件模式创建屏蔽字不会影响其父进程的屏蔽字
* 可以设置`umask`值以控制创建文件的默认权限
* Unix系统提供了`umask`命令
	* `-S`可以查看文件的默认权限

----

## chmod、fchmod和fchmodat函数

这三个函数可以更改现有文件的访问权限

### 函数定义

```c
#include <sys/stat.h>

int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);
// 成功返回0，出错返回-1
```

### 参数说明

`mode`参数就是那9个文件权限加上下面增加的这6个：

1. `S_ISUID`：执行时设置用户ID
2. `S_ISGID`：执行时设置组ID
3. `S_ISVTX`：保存正文（粘着位）
4. `S_IRWXU`：用户读写执行
5. `S_IRWXG`：组读写执行
6. `S_IRWXO`：其他读写执行

fchmodat函数在下面两种情况和chmod函数一样的：

* pathname是绝对路径的时候
* fd等于AT_FDCWD，且pathname是相对路径的时候
* 否则`fchmodat`计算相当于打开目录`fd`的`pathname`
* `flag`参数为`AT_SYMLINK_NOFOLLOW`时，`fchmodat`不会跟随符号链接

### 细节说明

 * 使用这个函数修改一个文件的权限时，**进程的有效用户ID**必须等于这个**文件的所有者ID**，或者这个进程具有超级用户权限
 * 粘着位：在下一个小结说明
 * 在下面两种条件下`chmod`会自动清除两个权限位：

 	1. 有些系统对粘着位有特殊的含义，在这些系统上如果没有超级管理员权限就去设置**普通文件**的粘着位的话，那么该函数会自动关闭`粘着位`
 	2. 新创建的文件的组ID可能不是调用进程所属的组时，设置组ID位会被自动关闭

----

## 粘着位

在Unix还没有使用请求分页式技术的时候，当一个可执行程序文件设置了粘着位(S_ISVTX)后，那么该改程序执行到它终止，程序正文部分的一个副本会保存在交换区中，这样可以在下次启动时较快的载入内存。
加载会变快的原因：通常的unix文件系统中，文件的各数据块都是随机存放的，而交换区是被作为一个连续文件来处理的
但是现在较新的unix系统都配置了虚拟存储系统以及快速文件系统，所以不再使用这种技术。

现在的系统扩展了粘着位的使用范围，可以对目录设置。如果一个目录设置了粘着位，只有对该目录具有**写权限**的用户且具有下列条件之一，才能**删除或重命名**该目录下的文件：

* 拥有此文件
* 拥有此目录
* 是超级用户

典型的目录有：`/tmp`和`/var/tmp`。任何用户都可以在这两个目录中创建文件，但是用户不能删除不属于自己的文件

----

## chown、fchown、fchownat和lchown

这几个函数用来修改文件的用户ID和组ID。

### 函数定义

```c
#include <unistd.h>

int chown(const char *pathname, uid_t owner, git_t group);
int fchown(int fd, uid_t owner, gid_t group);
int fchownat(int fd, const char *pathname, uid_t owner, gid_t group, int flag);
int lchown(const char *pathname, uid_t owner, gid_t group);
// 成功返回0，出错返回-1
```

### 细节说明

* 当非root用户修改了文件的所有者或所属组，并成功返回时，则文件的**设置组ID位**和**设置用户ID位**被自动关闭
* 若`_POSIX_CHOWN_RESTRICTED`（`chown`是否受限）对指定文件生效的话
  * 只有超级用户进程可以更改该文件的**用户ID**
  * 进程拥有此文件（进程的有效用户ID等于文件的用户ID），参数`owner`等于-1或文件的用户ID，并且参数`group`等于进程的有效组ID或者附属组ID之一的话，可以修改该文件的**组ID**

----

## 文件长度

* `stat`中的`st_size`字段表示文件的长度，以字节为单位。**这个字段只对普通文件、目录和符号链接有意义**
  * 对于目录来说，这个字段一般是一个数字（16、32）的整数倍
  * 对于符号链接来说，这个字段是链接地址的字节数。地址的长度是不包含结束符的
  * FreeBSD、MaxOS和Solaris系统对管道也定义了此字段，用于表示可以从管道读取到的字节数

`stat`中还提供了`st_blksize`和`st_blocks`两个字段

* `st_blksize`：表示对文件进行IO处理时，比较合适的块长度
* `st_blocks`：表示所分配的512字节块的块数。块的长度不一定都是512个字节，每个实现都不一样

----

## 文件截断

有时需要将文件尾端的数据截去一部分，有两种方式：

* 在打开文件时，使用`O_TRUNC`标志
* 使用`truncate`和`ftruncate`函数

### truncate和ftruncate函数定义

```c
#include <unistd.h>

int truncate(const char *pathname, off_t length);
int ftruncate(int fd, off_t length);
// 成功返回0，出错返回-1
```

* 当一个文件的长度大于`length`的时候，这文件长度会截断为`length`，`length`之后的数据将无法访问
* 如果小于`length`的时候，则会将文件扩展到`length`，扩展的地方以0填补，会创建出一个空洞文件

----

## 文件系统

![](https://raw.githubusercontent.com/cxldada/imgs/master/202403121149894.png)

* 每个`i节点`中都会保存一个链接数的信息（通常是`stat`结构中的`st_nlink`成员），当链接数为0时，才可以删除文件。这就解释了为什么linux中删除文件是`unlink`而不是`delete`的原因。
* `LINK_MAX`制定了一个文件链接数的最大值
* `stat`结构中的`st_nlink`的类型为`nlink_t`，这种链接类型称为硬链接。也就是链接文件**直接指向目标文件的`i节点`**
* 另外一种链接类型称为**符号链接**。链接文件的实际内容是**指向目标文件的名字(完整路径)**。也就是**软链接**
* 任何一个叶目录的链接计数总是2，一个是该目录文件本身，一个是该目录文件中的 **.** 目录
* 父目录中的每个子目录都会使该父目录的链接计数增加1，因为每个子目录中都有 **..** 目录

----

## link、linkat、unlink、unlinkat和remove函数

而已使用`link`和`linkat`函数创建指向现有文件的链接。（硬链接，直接指向文件的`i结点`）

### `link`和`linkat`函数定义

```c
#include <unistd.h>
int link(const char *existingpath, const char *newpath);
int linkat(int efd, const char *existingpath, int nfd, const char *newpath, int flag);
// 成功返回0，出错返回-1
```

#### 参数说明

`linkat`函数需要注意的地方：

* `existingpath`和`newpath`为绝对路径时，则两个`fd`参数被忽略
* `existingpath`和`newpath`为相对路径时，则相对路径是按照两个`fd`所指向的目录计算的
* `efd`和`nfd`为`AT_FDCWD`时，则相对于当前目录进行计算
* `flag`参数如果为`AT_SYMLINK_NOFOLLOW`时，就创建符号链接的链接，而不是符号链接所指向的目标文件的链接

#### 细节说明

* 如果`newpath`已经存在，则出错
* 只创建`newpath`中的最后一个分量，路径中的其他部分应该已经存在
* 创建新目录项和增加连接计数是一个原子操作
* 很多文件系统不允许创建对于目录的硬链接，因为可能会造成引用循环。

### `unlink`和`unlinkat`函数定义

```c
#include <unistd.h>

int unlink(const char *pathname);
int unlinkat(int fd, const char *pathanme, int flag);
// 成功返回0，失败返回-1
```

#### 参数说明

`flag`参数为`AT_REMOVEDIR`时，`unlinkat`函数可以类似于`rmdir`一样删除目录。

#### 细节说明

* 这两个函数会将`pathname`所引用的文件的链接计数减1，当链接计数为0时，文件内容才会被真正的清除
* 解除对文件的链接，需要对包含该文件的目录具有**写和执行权限**
* 当有进程打开了该`pathname`指向的文件时，就算链接计数为0了，数据内容也不会删除。只有当引用该文件的进程数为0，且链接计数为0时才会删除这个文件的数据内容
* 可以使用上面这条性质在进程中创建临时文件。用`open`或`creat`创建一个文件后，立即调用`unlink`，此时文件的内容不会被删除，直到进程结束，该临时文件才会被真正的删除
* 如果参数指向的是一个符号链接，则删除的是符号链接本身。没有一个函数能够删除符号链接所引用的文件

### `remove`函数定义

```c
#include <stdio.h>

int remove(const char *pathname);
// 成功返回0，出错返回-1
```

对于文件，`remove`相当于unlink，对于目录，`remove`相当于`rmdir`

----

## `rename`和`renameat`函数

使用这两个函数可以修改文件或目录的名字

### 函数定义

```c
#include <stdio.h>

int rename(const char *oldname, const char *newname);
int renameat(int oldfd, const char *oldname, int newfd, const char *newname);
// 成功返回0，出错返回-1
```

### 参数和细节说明

根据`oldname`指向的是文件、目录或是符号链接，有几种情况需要说明，以及`newname`已存在的情况：

1. `oldname`是一个文件：
   * 如果`newname`已经存在，则它不能是一个目录
   * 如果`newname`已经存在且不是一个目录，则先删除该**目录项(不是目录)**然后再重命名为`newname`，注意目录项是该文件的引用，不是文件的数据内容。所以调用进程要对包含`oldname`和`newname`的目录，必须具有写和执行权限。因为要删除再重命名嘛
2. `oldname`是一个目录的情况：
   * 如果`newname`已经存在，则它必须是一个目录，而且该目录应该是一个空目录
   * 如果`newname`已经存在，也是先删除再重命名
   * `newname`不能包含`oldname`作为其路径前缀
3. 如果`oldname`或者`newname`引用符号链接，则处理的是符号链接本身，而不是符号链接所指向的目标文件
4. 不能对 **.** 和 **..** 重命名
5. 如果`oldname`和`newname`引用的是同一个文件，则函数不做任何处理并成功返回

## 创建和读取符号链接
符号链接也称为软链接，主要是用来解决硬链接的两个局限：
* 源文件与硬链接必须属于同一文件系统
* 硬链接不能指向目录

用下面两个函数创建符号链接

```c
#include <unistd.h>
int symlink(const char *actualpath,const char *sympath);
int symlinkat(const char *actualpath,int fd,const char *sympath);
// 成功返回0，出错返回-1
```

并不要求`actualpath`指向的文件或目录一定存在，而且源文件也不必与符号链接文件在同一个文件系统中

由于open函数总是跟随符号链接访问目标文件，所有可以用下面两个函数获取符号链接本身的内容

```c
#include <unistd.h>
ssize_t readlink(const char *restrict pathname,char *restrict buf,size_t bufsize);
ssize_t readlinkat(int fd,const char *restrict pathname,char *restrict buf,size_t bufsize);
// 成功返回读取的字节数，出错返回-1
```

这两个函数组合了open、read和close的所有操作。

**注意： buf中返回的内容不以null字符终止，所以使用时需要注意**

----

## 修改文件访问时间和修改时间

每个文件都维护了3个时间属性：

1. 文件最后被访问的时间：`st_atime`
2. 文件最后被修改的时间：`st_mtime`
3. `i节点`最后修改的时间：`st_ctime`

```c
#include <sys/stat.h>
int futimens(int fd,const struct timespec times[2]);
int utimensat(int fd,const char *paht,const struct timespec times[2],int flag);
// 成功返回0，出错返回-1
```

times是一个数组，第一个元素表示要设定的**访问时间**，第二个元素表示要设定的**修改时间**

times可以按照下面四中方式指定

1. 如果times是空指针，则访问时间和修改时间都设置为当前时间
2. 如果times数组中的两个元素，任意一个元素的`tv_nsec`字段值为`UTIME_NOW`，相应的时间戳就设置为当前时间，忽略tv_sec字段
3. 如果times数组中的两个元素，任意一个元素的`tv_nsec`字段值为`UTIME_OMIT`，相应的时间戳保持不变，忽略tv_sec字段
4. 如果两个元素的tv_nsec不属于第2,3中情况，按照元素值进行设置

执行这些函数要求的优先权也取决于times的值

1. 如果times是空指针，或者任意一个tv_nsec字段为`UTIME_NOW`，则进程的**有效用户ID**必须等于**文件所有者ID**，进程必须对文件具有**写权限**，或者进程是root进程
2. times非空时，任意一个tv_nsec字段都不是`UTIME_NOW`或者`UTIME_OMIT`，则进程的**有效用户ID**必须等于该文件的所有者ID，或者进程是root进程。对文件只有写权限是不够的
3. times非空，两个tv_nsec都是UTIME_OMIT，就不执行任何权限检测

flag参数，是用来控制是否跟随符号链接文件的，指定了`AT_SYMLINK_NOFOLLOW`就不跟随。

还有一个函数是XSI的扩展

```c
#include <sys/time.h>
int utimes(const char *pathname,const struct timeval times[2]);
// 成功返回0，失败返回-1
struct timeval {
    time_t tv_sec;	// 秒
    long tv_usec;	// 纳秒
}
```

这个函数的作用和上面两个函数相同。

同时我们不能修改文件的`i节点`访问时间，也就是`st_ctime`。我们在`utimes`函数的时候回自动更新这个字段

## 创建目录的函数

创建目录的函数

```c
#include <sys/stat.h>
int mkdir(const char *pathname,mode_t mode);
int mkdirat(int fd,const char *pathname,mode_t mode);
// 成功都返回0，出错返回-1
```

要点：

* 这两个函数会创建一个空目录，其中 **.** 和 **..** 目录会自动创建
* 所指的文件访问权限mode由进程的文件模式创建屏蔽字修改
* 有一个常见的错误，就是没有设置执行位权限。这会导致无法访问目录中的文件名
* 新目录的用户ID和组ID会根据上面第6小结内容进行设置。组ID的设置是要看系统实现的

----

## 删除目录的函数

删除一个空目录(注意是空目录)

```c
#include <unistd.h>
int rmdir(const char *pathname);
// 成功返回0，出错返回-1
```

要点：

* 如果要删除的目录的链接计数为0，并且没有其他进程打开该目录，则释放由该目录所占用的空间
* 如果链接计数为0了，但是有进程还在占用该目录，则在此函数返回前删除最后一个链接及 **.** 和 **..** 项
* 如果出现上一项中的情况，那么此目录中不能在创建新的文件。因为rmdir要确保删除的目录必须是空的

----

## 读取目录的函数

为了防止文件系统产生混乱，只有内核才可以写目录。

一个目录的写权限位和执行权限位决定了在该目录中是否可以创建或删除文件，并不代表可以写目录本身

很多实现会阻止使用`read`函数读目录，所以提供了下面的函数

### 函数定义

```c
#include <dirent.h>
DIR *opendir(const char *pathname);
DIR *fdopendir(int fd);
// 成功则返回指针，出错返回NULL

struct dirent *readdir(DIR *dp);
// 成功则返回指针，出错或者已经到目录尾部，则返回NULL

oid rewinddir(DIR *dp);
int closedir(DIR *dp);
// 成功返回0，出错返回-1

long telldir(DIR *dp);
// 与dp关联的目录中的当前位置

void seekdir(DIR *dp,long loc);

// 对于dirent结构。标准要求实现至少包含下面两个成员
ino_t d_ino;	// i-node节点编号
char d_name[];	// 文件名

// DIR结构是一个内部结构，类似于FILE结构。FILE结构后面学习标准IO时会说明
```

这一套函数由这个[实例](https://github.com/cxldada/Exercise/blob/master/c/opeator_dir_func.c)讲解比较方便

## 修改当前工作目录的函数

每个进程都有一个当前工作目录，这个目录是所有相对目录搜索的起点。

可以下面的函数修改进程的当前工作目录

### 函数定义

```c
#include <unistd.h>
int chdir(const char *pathname);
int fchdir(int fd);
// 成功返回0，出错返回-1
```

### 细节说明
这两个函数只会影响调用进程本身，不会影响其他进程

## 获取当前工作目录完整路径的函数

```c
#include <unistd.h>
char *getcwd(char *buf,size_t size);
// 成功返回char字符串，出错返回NULL
```

### 细节说明

buf的size大小是包含了最后的结束空字符的

----

## 设备特殊文件

* 每个文件系统所在的存储设备都由其主、次设备号表示。
* 设备号用基本系统数据类型`dev_t`记录
* 我们可以使用`major`和`minor`来访问主、次设备号
* 系统中与每个文件名关联的`st_dev`值是文件系统的设备号，改文件系统包含了这一文件名以及与其对应的`i结点`
* 只有字符特殊文件和块特殊文件才有`st_rdev`值。

[代码示例](https://github.com/cxldada/Exercise/blob/master/c/st_dev_test.c)

----
