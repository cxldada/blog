---
title: 3. 标准I/O库
date: 2020-07-18 08:22:51
draft: false
categories: 
- APUE
---

标准IO库是一个方便用户使用的库，他让用户不用关心缓冲区、块长度等问题。但是我们还是认真学习标准IO库，因为如果不深入了解标准IO库的某些特点，那么在使用的时候可能会带来一些小麻烦。

<!--more-->

## 流和FILE对象

在之前学习文件IO的时候，可以看得出来所有的IO函数都是以打开的文件描述符为基础来进行操作的。而标准IO库则是围绕着流来进行相关IO操作的。当我们使用标准I/O打开或创建一个文件时，我们已使一个流与一个文件相关联

由于有多种字符集(ASCII、国际字符集等)，每种字符集的字节宽度也是不一样的，有的一个字符占一个字节（单字节），有的一个字符占多个字节（宽字节）。标准IO库为了满足各种字符集，所以流是可以设置单字节或宽字节的，成为**流的定向**。

当一个流刚被创建的时候是没有定向的。第一次在一个未定向的流上使用单字节I/O函数或是多字节I/O函数的时候，流就会被自动的设置为字节定向或宽定向

----

## 改变流的定向

只有两个函数可以改变流的定义：

* `freopen`函数清除一个流的定向
* `fwide`函数为一个未定向的流设置定向

### 函数定义
```c
#include <stdio.h>
#include <wchar.h>

int fwide(FILE *fp, int mode);
// 返回值：
// 若流是宽定向则值 > 0
// 若流是字节定向则值 < 0
// 若流未定向则值 = 0
```

### 参数说明

`mode`参数：

* 如果mode为负值，则将流设置为字符定向
* 如果mode为正值，则将流设置为宽定向
* 如果mode为0，则返回当前流的状态，并不改变流的定向

### 细节说明

* `fwide`函数并不能改变以定向的流。
* `fwide`函数没有出错返回
* 由于没有出错返回，所有一般在使用这个函数前现将`errno`设置为0，然后在调用后来检测`errno`的值

## 标准输入、标准输出和标准错误

每个进程都预定义了3个流，这三个流可以直接使用，他们在`<stdio.h>`中定义为：`stdin`、`stdout`和`stderr`
在文件I/O那边文章中提到过，可以使用`STDIN_FILENO`、`STDOUT_FILENO`和`STDERR_FILENO`这三个文件描述符引用

## 缓冲

为了减少调用`read`和`write`的次数，标准I/O提供了自己的缓冲机制。
一共提供了三种缓冲机制：
1. 全缓冲。只有在写入的数据填满缓冲区的时候才会执行系统I/O操作。
	* 当第一次在一个流上执行I/O操作的时候，相关的标准I/O函数会调用`malloc`函数分配所需的缓冲区。一般全缓冲的缓冲区大小由`<stdio.h>`中的`BUFSIZ`指定。
	* 一般用于处理文件的流都是全缓冲
2. 行缓冲。当输入换行符的时候就会执行系统I/O操作。有时就算没有输入换行符，但是缓冲区已满也会触发系统I/O操作。
	* 任何时候只要通过标准I/O库从一个不带缓冲的流，或者一个行缓冲的流得到数据时，就会冲刷所有的行缓冲输出流
	* 几乎所有的终端都是使用行缓冲
3. 不带缓冲。不提供缓冲机制。
	* 标准错误就是不带缓冲的流，这样就可以尽快的将数据显示出来

----

## 冲刷缓冲区

有时希望全缓冲或行缓冲的流在缓冲区还未满的时候就执行系统I/O，可以使用`fflush`函数冲刷流

### 函数定义

```c
#include <stdio.h>

int fflush(FILE *fp);
// 成功返回0，出错返回EOF
```

### 细节说明

如果`fp`参数为`NULL`，那么会冲刷所有已打开的流

## 修改流的缓冲类型

当打开一个流的时候，系统设置默认的缓冲类型，如果我们不喜欢默认的缓冲类型，可以使用下面两个函数修改

### 函数定义

```c
#include <stdio.h>

void setbuf(FILE *restrict fp, char *restrict buf);
int setvbuf(FILE *restrict fp, char *restrict buf, int mode, size_t size);
// 成功返回0，出错返回非0
```

### 参数说明
`setbuf`函数，只有两种情况：
* `buf`不为空，则设置流的缓冲区，但是由系统决定是行缓冲还是全缓冲，缓冲区的大小为`BUFSIZ`（定义在stdio.h中）
* `buf`为空，则设置流为不带缓冲的

`setvbuf`函数的`mode`参数：
* `_IOFBF`：全缓冲
	* 如果`buf`非空，这缓冲区长度为`size`
	* 如果`buf`为空，则系统设置默认的大小
* `_IOLBF`：行缓冲
	* 如果`buf`非空，这缓冲区长度为`size`
	* 如果`buf`为空，则系统设置默认的大小
* `_IONBF`：不带缓冲
	* 忽略`buf`和`size·参数

### 细节说明
* 如果`buf`是在一个函数内定义的自动变量，那么在离开函数的时候应该关闭流，不然离开函数后变量会被清除。
* 有些系统实现会在缓冲区中存放一些操作信息，所有实际可用的缓冲区大小会小于`size`
* 最好是使用系统默认分配的缓冲区和缓冲区大小，这样在关闭流的时候，缓冲区也会由系统自动回收

----

## 打开流

有三种方式可以打开流

### 函数定义

```c
#include <stdio.h>
FILE *fopen(const char *restrict pathname, const char *restrict type);
FILE *freopen(const char *restrict pathname, const char *restrict type, FILE *restrict fp);
FILE *fdopen(int fd, const char *type);
// 成功返回文件指针，出错返回NULL
```

### 参数说明

`type`参数指定对I/O流的读写方式（`b`表示二进制文件）：
* `r`或`rb`：读打开
* `w`或`wb`：写打开。文件打开时会把文件截断为0。若无文件则可以创建文件
* `a`或`ab`：追加打开。每次写都在文件尾部操作
* `r+`或`r+b`或`rb+`：读写打开
* `w+`或`w+b`或`wb+`：读写打开，打开时会把文件截断为0
* `a+`或`a+b`或`ab+`：读写打开，每次写都在文件尾部。若无文件则可以创建文件

`fdopen`函数的`type`参数有两点区别：

1. 所有的截断操作都不会生效。因为是否截断文件要看描述符的打开方式中有没有`O_TRUNC`标志
2. 所有的追加写操作也不会创建文件。都有文件描述符了创建个屁啊。

### 细节说明

这三个函数的区别如下：
1. `fopen`函数打开指定路径的文件
2. `freopen`函数在一个指定的流上打开一个指定的文件。
	* 如果该流已打开，则先关闭。
	* 如果流已定向，则使用`freopen`清除该定向
	* 一般用于将一个指定的文件打开为一个预定义的流。比如把预定义的标准输入、标准输出或标准错误指向指定的文件。
3. `fdopen`函数是把一个文件描述符与一个标准I/O流结合。
	* 常常用于由创建管道和网络通信函数返回的描述符。因为这些特殊类型的文件描述符无法用`fopen`打开

**如果多个进程用标准I/O追加写的方式打开同一个文件时，确保每个进程的数据都可以正确的写入文件**

当以读写的方式打开一个文件时，有以下限制：
* 如果中间没有`fflush`、`fseek`、`fsetpos`或`rewind`，则输出后面不能直接跟随输入。
* 如果中间没有`fseek`、`fsetpos`或`rewind`，或者一个输入操作没有到达文件尾端，则在输入操作之后不能直接跟随输出

使用标准I/O函数创建的文件，我们无法确定文件的访问权限位。`POSIX`要求各个实现使用 `S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH ` 来设置新创建的文件。我们可以通过`umake`来限制这些权限

除非流引用终端设备，否则系统默认流是**全缓冲**的，如果是终端设备，则流是**行缓冲**的

## 关闭流

```c
#include <stdio.h>

int fclose(FILE *fp);
// 成功返回0，出错返回EOF
```

关闭时会冲洗缓冲中的输出数据。缓冲区的所有数据都**会被丢弃**。如果是自动分配的缓冲区，系统还会释放改缓冲区

## 读写流

一旦打开了流，就有三种不同类型的非格式化I/O操作：
* 每次处理一个字符的I/O
* 每次处理一行字符的I/O
* 直接I/O或叫二进制I/O

还可以使用两个格式化I/O函数：
* `printf`
* `scanf`

### 每次处理一个字符的I/O

#### 输入函数定义

```c
#include <stdio.h>

int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);
// 成功返回下一个字符，若已达到文件尾或出错，返回EOF
```

#### 细节说明

* `getchar`相当于`getc(stdin)`。
* `getc`函数可以是宏，而`fgetc`不能是宏，这就造成这两个函数的一些区别：
	1. `getc`的参数不能是具有副作用的表达式，因为它会被多次计算
	2. 因为`fgetc`是一个函数，所有可以得到其函数地址，就能当做参数传给其他函数，而`getc`不可以。
	3. 调用`fgetc`的时间比`getc`长，因为调用函数的时间通常比调用宏的时间长

返回值为整数的原因：

* 这样可以返回所有可能的字符值再加上一个已出错或已达文件尾端（`EOF`）的指示值。因为在`<stdio.h>`中`EOF`通常被定义为`-1`

因为返回值包含了到文件尾部和EOF错误，所以可以使用下面两个函数确定到底是哪一种情况

```c
#include <stdio.h>

int ferror(FILE *fp);
int feof(FILE *fp);
// 为真返回非0，为假返回0

void clearerr(FILE *fp);
```

大多数实现中，都会为每个流维护两个标志：
* 出错标志
* 文件结束标志

调用`clearerr`可以清楚这两个标志

#### 回送字符的函数

```c
#include <stdio.h>

ungetc(int c,FILE *fp);
// 成功返回c，出错返回EOF
```

* 回送的字符，在下去读取时会这取出。
* `ungetc`调用成功时，会清除文件结束标志。所以即便是到了文件尾部也可以会送字符，并在读取该字符后，下一个字符一定是结束符

#### 输出函数定义

```c
#include <stdio.h>

int putc(int c, FILE *fp);
int fputc(int c, FILE *fp);
int putchar(int c);
// 成功返回c，出错返回EOF
// 与读取函数基本相同，putc可以实现为宏定义，而fputc必须是函数定义
```

### 每次一行的IO

```c
#include <stdio.h>

char *fgets(char *restrict buf, int n, FILE *restrict fp);
char *gets(char *buf);
// 成功返回buf，若已到文件尾或出错，返回NULL

int fputs(const char *restrict buf, FILE *restrict fp);
int puts(const char *str);
// 成功返回非负值，出错返回EOF
```

#### 细节说明

* 两个函数都是将数据送入指定的缓冲区，`gets`是从标准输入读取，而`fgets`是从指定的流读取
* `fgets`会保留换行符，`gets`函数不会保留
* `fputs`中的`buf`要以`null`结尾，但是`null`不会写出。
* `puts`函数会自动加入一个换行符
* 建议不要使用`gets`函数，因为它无法指定缓冲区的大小，容易造成数据超长的bug。

### 二进制I/O

```c
#include <stdio.h>

size_t fread(void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
size_t fwrite(const void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
// 返回读或写的对象数量
```



#### 参数说明

* `ptr`是指向存放结构体的指针
* `size`是单个结构的大小
* `nobj`是需要读取几个结构体
* `fp`就是IO

#### 细节说明

* 当达到文件尾端或出错，`fread`返回的值可能会小于`nobj`，这个时候还是要调用`ferror`或`feof`的函数来判断到底是哪种情况

* 二进制IO只能用于读在同一系统上已写的数据，原因是：
  1. 在一个结构中，同一成员的偏移量可能随编译程序和系统的不同而不同(由于不同的对齐要求)。
  2. 用来存储多字节整数和浮点值的二进制格式在不同的系统结构间也可能不同

## 定位流

在UNIX环境中共有三种方式来定位流：

1. `ftell`和`fseek`函数。他们是最早用来定位流的函数，但是它们都假定文件的位置可以存放在一个长整型中
2. `ftello`和`fseeko`函数。这是`Single UNIX Specification`引入的两个函数。他们使用`off_t`代替了长整型
3. `fgetpos`和`fsetpos`函数。这是`ISO C`引入的。他们使用一个抽象数据类型`fpos_t`记录文件的位置。这种数据可以根据需要定义一个足够大的数。

**如果需要将程序移植到非UNIX系统上运行的话，最好使用`fgetpos`和`fsetpos`函数。**

### ftell系列函数定义

```c
#include <stdio.h>
long ftell(FILE *fp);
// 成功返回当文件文件位置，出错返回 -1L

int fseek(FILE *fp, long offset, int whence);
// 成功返回0，出错返回-1

void rewind(FILE *fp);
// 将流设置到文件的起始位置
```

### 参数说明

- `whence`参数与`lseek`函数相同可以取：`SEEK_SET、SEEK_CUR、SEEK_END`
- 对于一个文本文件来说，如果是用`fseek`的话，`whence`必须是`SEEK_SET`，而且`offset`只能是`0`或`ftell`所返回的值。对于二进制文件就没有这种限

### ftello函数系列

```c
#include <stdio.h>
off_t ftello(FILE *fp);
// 成功返回当文件文件位置，出错返回(off_t)-1

int fseeko(FILE *fp,off_t offset,int whence);
// 成功返回0，出错返回-1
```

### fgetpos函数系列

```c
#include <stdio.h>
// 下面这两个函数是ISO C提供的。如果想要移植到非unix系统中，最好使用它们
int fgetpos(FILE *restrict fp,fpos_t *restrict pos);
int fsetpos(FILE *fp,const fpos_t *pos);
// 成功返回0，出错返回非0
```

-----

## 格式化I/O

### 格式化输出

格式化输出有5个函数

```c
#include <stdio.h>

int printf(const char *restrict format, ...);
int fprintf(FILE *restrict fp, const char *restrict format, ...);
int dprintf(int fd, const char *restrict format, ...);
// 成功返回输出的字符数，出错返回负值

int sprintf(char *restrict buf, const char *restrict format, ...);
int snprintf(char *restrict buf, size_t n, const char *restrict format, ...);
// 若缓冲区足够大，则返回存入数组的字符数，出错返回负值
```

这几个函数的区别：

* `printf`将格式化数据写到**标准输出**
* `fprintf`写到**指定的文件流**
* `dprintf`写到**指定的文件描述符**
* `sprintf`和`snprintf`写到字符数组中。
  * 这两个函数会在数组尾部自动加入一个`null`字节，但是返回值中不计算该字符
  * `sprintf`是不安全的，因为它有可能会导致缓冲区溢出，它无法设置写入的大小，而`snprintf`就相对安全
  * `snprintf`函数会将超过长度的数据丢弃

#### 参数说明

`format`参数由4个可选的部分组成：

```tex
%[flags][fldwidth][precision][lenmodifier]convtype
%[标记][最小字段宽度][精度][长度修饰]转换类型
```

1. `flags` -- 标记类型：
   * `'`：将整数按照千位分组
   * `-`：在字段内左对齐输出
   * `+`：总是显示带符号转换的正负号
   * 空格：如果第一个字符不是正负号，则在前面加上一个空格
   * `#`：指定另一种转换形式，例如十六进制，加`0x`前缀
   * `0`：添加前导0进行填充

2. `fldwidth` -- 指定最小字段宽度，如果参数字符不够，则多余的字符位置用空格填充(也可以是0填充，看flags标志),是一个的非负十进制数或星号`*`

3. `precision` -- 精度：
   * 整形转换后最少输出数字位数
   * 浮点数转换后小数点后的最少位数
   * 字符转换后最大字节数
   * 精度是`.`后面跟随一个非负十进制数或者一个星号`*`
4. `lenmodifier` -- 长度修饰。可选的值有：
   * `hh`：将相应的参数按照`signed`或`unsigned char`类型输出
   * `h`：将相应的参数按照`signed`或`unsigned short`类型输出
   * `l`：将相应的参数按照`signed`或`unsigned long`或等宽字符类型输出
   * `ll`：将相应的参数按照`signed`或`unsigned long long`类型输出
   * `j`：`intmax_t`或`uintmax_t`
   * `z`：`size_t`
   * `t`：`ptrdiff_t`
   * `L`：`long double`
5. `convtype` -- 转换类型
   * `d、i`：有符号十进制
   * `o`：无符号八进制
   * `u`：无符号十进制
   * `x、X`：无符号十六进制
   * `f、F`：双精度浮点数
   * `e、E`：指数格式双精度浮点数
   * `g、G`：根据转换后的值解释`f、F、e、E`
   * `a、A`：十六进制指数格式双精度浮点数
   * `c`：字符（若带长度修饰符`l`，表示宽字符）
   * `s`：字符串
   * `p`：指向`void`的指针
   * `n`：到目前为止函数已输出的字符的数目
   * `%`：一个`%`字符
   * `C`：宽字符 == `lc`
   * `S`：宽字符串 == `ls`

#### 5个变种printf函数

```c
#include <stdarg.h>
#include <stdio.h>

int vprintf(const char *restrict format, va_list arg);
int vfprintf(FILE *restrict fp, const char *restrict format, va_list arg);
int vdprintf(int fd, const char *restrict format, va_list arg);
// 成功返回输出的字符数，出错返回负值

int vsprintf(char *restrict buf, const char *restrict format, va_list arg);
// 成功返回存储数组的字符数，出错返回负值

int vsnprintf(char *restrict buf, size_t n, const char *restrict format, va_list arg);
// 若缓冲区够大，返回存入数组的字符数，出错返回负值
```



### 格式化输入

有几个函数用来处理格式化输入

```c
#include <stdio.h>

int scanf(const char *restrict format, ...);
int fscanf(FILE *restrict fp, const char *restrict format, ...);
int sscanf(const char *restrict buf, const char *restrict format, ...);
// 成功返回赋值的输入项数，出错或在转换前达到文件尾，返回EOF
```

#### 参数说明

`fromat`参数由3个可选部分组成：

```
%[*][fldwidth][m][lenmodifier]convtype
```

1. `*`：用来抑制转换，输入的值转换后不存入参数
2. `fldwidth`最大宽度
3. `lenmodifier`和格式化输出相同
4. `convtype`和格式化输出相同，但有一点区别：
   1. 作为一种选项，输入中带符号的可赋予无符号类型
5. `m`是赋值分配符
   1. 可以用于`%c`、`%s`和`%[`转换符，迫使内存缓冲区分配空间以接纳转换字符串
   2. 接收参数必须是指针地址，分配的缓冲区地址必须复制给该指针

#### 3个变种scanf函数

```c
#include <stdarg.h>
#include <stdio.h>

int vscanf(const char *restrict format, va_list arg);
int vfscanf(FILE *restrict fp, const char *restrict format, va_list arg);
int vsscanf(const char *restrict buf, const char *restrict format, va_list arg);
// 成功返回赋值的输入项数，出错或在转换前达到文件尾，返回EOF
```



## 获取与流相关联的描述符

```c
#include <stdio.h>
int fileno(FILE *fp);
// 返回与该流相关联的文件描述符
```

## 临时文件

ISO C标准提供了两个函数用来创建临时文件

```c
#include <stdio.h>

char *tmpnam(char *ptr);
// 返回指向唯一路径名的指针

FILE *tmpfile(void);
//  成功返回文件指针，出错返回NULL
```

### 细节说明

* `tmpnam`函数产生一个与现有文件名不同的一个有效路径名字符串
  * 每次调用该函数都会产生一个不同的路径名
  * 最多调用`TMP_MAX`次（定义在`<stdio.h>`中）
  * 如果`ptr`指向`NULL`，则所产生的路径名存放在一个静态区中，函数返回指向该静态区的指针（后续调用该函数会重写静态区）
  * 如果`ptr`不为`NULL`，则它应该指向长度至少为`L_tmpnam`个字符的数组（定义在`<stdio.h>`中）。`ptr`有作为函数值返回
* `tmpfile`创建以类临时二进制文件（类型为`wb+`），关闭调用进程会自动删除这种文件

[程序实例](https://github.com/cxldada/Exercise/blob/master/c/tempfile.c)

### 相同函数

`Single UNIX specification`提供了另外两个函数

```c
#include <stdlib.h>

char *mkdtemp(char *template);
// 成功返回指向目录名的指针，出错返回NULL

int mkstemp(char *template);
// 成功返回文件描述符，出错返回-1
```

#### 细节说明

* `mkdtemp`用于创建临时目录。`mkstemp`用于创建临时文件
* `template`参数的是一个路径名，但是该路径名的后6位必须是`XXXXXX`
* `mkdtemp`创建的临时目录的权限位：`S_IRUSR | S_IWUSR | S_IXUSR`。除此之外还可以使用文件模式创建屏蔽字进一步限制
* `mkstemp`创建的文件权限位：`S_IRUSR | S_IWUSR`
* `mkstemp`创建的临时文件与`tmpfile`创建的临时文件不同，它不会被自动删除。如果需要，必须手动删除
* `template`参数最好是定义成字符数组。如果是数组的话，该数组的空间是分配在栈上的，所以`mkstemp`函数可以进行读写操作，如果使用字符指针的话，只有指针变量是在栈上，而实际指向的内容存在可执行文件的只读段中，所以`mkstemp`不能够修改会导致程序崩溃异常

**注意：**

使用`tmpnam`和`tmpfile`至少有一个缺点：在返回唯一的路径名和用该名字创建文件之间存在一个时间窗口，在这个时间窗口中，另一个进程可以用相同的名字创建文件。

[实例代码](https://github.com/cxldada/Exercise/blob/master/c/mkstemp_test.c)



----

## 内存流

内存流相较于普通的文件流主要是在于普通文件流有底层文件，而内存流没有。普通的文件流需要将缓冲区的内容写到磁盘中，而内存流则是缓冲区与主存之间的数据交流。

```c
#include <stdio.h>
FILE *fmemopen(void *restrict buf,size_t size,const char *restrict type);
// 成功返回流指针，出错返回NULL
```

* 只有最后一个参数需要说明一下，它与`fopen`中的`type`是相同的，所以可以参数上面`fopen`函数的解释来使用
* 如果`buf`为空，则系统分配`size`大小的缓冲区，当关闭流时也会自动清除
* 虽然type的取值相同，但是功能细节还是有点小差别的，具体如下：
  * 如果以追写的方式打开内存流，当前文件位置设为缓冲的第一个null字节。如果缓冲区中不存在`null`字节，则当前位置设置为缓冲区结尾的后一个字节。不是追写方式打开的话，则当前位置设置为缓冲的开始位置。因为追写模式通过第一个null字节来确定数据的尾端
  * 内存流不适合处理二进制数据
  * 如果`buf`参数为`null`指针，则对打开的流进行读写操作是没有任何意义的
  * 任何时候需要增加流缓冲区中的数据量，以及调用`fclose、fflush、fseek、fseeko`以及`fsetpos`时都会在当前位置写入一个`null`字节

[实例代码](https://github.com/cxldada/Exercise/blob/master/c/memoopen.c)

### 同族函数

```c
#include <stdio.h>

FILE *open_memstream(char **bufp,size_t *sizep);
// 成功返回流指针，出错返回NULL

#include <wchar.h>

FILE *open_wmemstream(wchar_t **bufp,size_t *sizep);
// 成功返回流指针，出错返回NULL
```

### 细节说明

这两个函数与`fmemopen`的差别：

* 创建的流只能写打开
* 不能指定自己的缓冲区，但可以分别通过`bufp`和`sizep`参数访问缓冲区地址和大小
* 关闭流后需要自行释放缓冲区
* 对流添加字节会增加缓冲区大小

















































