---
title: 4.系统数据文件和信息
date: 2020-07-20 14:03:16
draft: false
categories: 
- APUE
---

由于历史原因，Unix中很多数据文件都是`ASCII`文本文件，可以使用标准IO读这些文件。但是，对于较大的系统，如果顺序扫描这些文件的话，效率会非常的低。因此我们需要使用非`ASCII`格式来存储这些数据文件，但仍要向其他文件格式提供接口。所以本章主要关注的是与这些**数据文件操作有关的接口**。还包括系统标识**函数、时间和日期函数**

<!--more-->

# 口令文件

口令文件记录了用户登录相关的数据，数据存放于`/etc/passwd`中

每个口令文件的数据结构存放在`<pwd.h>`中，数据结构如下：

```c
#include <pwd.h>

struct passwd {
    char *pw_name;		// 用户名
    char *pw_passwd;	// 加密口令
    uid_t pw_uid;		// 数值用户ID
    gid_t pw_gid;		// 数值组ID
    char *pw_gecos;		// 注释字段
    char *pw_dir;		// 初始工作目录
    char *pw_shell;		// 初始shell
    char *pw_class;		// 用户访问类(FreeBSD,MacOS)
    time_t pw_change;	// 下次更改口令时间(FreeBSD,MacOS)
    time_t pw_expire;	// 账户有效期时间(FreeBSD,MacOS)
};
```

口令文件中的内容就是`passwd`结构中的每一项用冒号分开来表示的，形式如下：

```
pw_name:pw_passwd:pw_uid:pw_gid:pw_gecos:pw_dir:pw_shell
```

关于口令文件的注意项：

* 会有一个root用户，它的用户ID和组ID是0
* 加密口令字段包含了一个占位符。如果加密口令放在一个人人可读的文件中是一个安全漏洞，所以实际的加密口令存放在另一个文件中
* 口令文件中的某些项可能是空。比如没有注释字段。
* shell字段包含了一个可执行文件名，它是用户登录时使用的shell，如果设置为`/dev/null`表示禁止该用户登录
* 为了阻止特定用户登录，除了`/dev/null`外，还可以使用`/bin/false`或`/bin/true`。如果系统提供了`nologin`命令也可以使用
* 在口令文件中应该会有一个`nobody`用户，任何人都可以使用该账户登录，但是这个账户的权限很低。他的`uid`和`gid`通常是`65534`
* 提供`finger`命令的某些UNIX系统支持注释字段中的附加信息，具体看命令详情

## 获取口令文件项的函数

```c
#include <pwd.h>

struct passwd *getpwuid(uid_t uid);
struct passwd *getpwnam(const char *name);
// 成功返回指针，失败返回NULL
```

这两个函数返回的`passwd`结构体通常是函数内部的静态变量，**只要调用任一相关函数，其内容就会被重写**

## 遍历口令文件项的函数

```c
#include <pwd.h>

struct passwd *getpwent(voi);
// 成功返回指针，出错或达到文件尾返回NULL

void setpwent(void);
void endpwent(void);
```

* `getpwent`函数用于获取口令文件中的下一个记录项。
* `setpwent`函数用于将反绕它所使用的文件，就是使下次获取记录项时，为口令文件中的第一项
* `endpwent`函数用于关闭文件。因为`getpwent`函数知道什么打开文件，但它不知道什么时候关闭

实例：使用这三个函数实现上一节中的`getpwnam`函数

```c
#include <pwd.h>
#include <stddef.h>
#include <string.h>

struct passwd *getpwnam(const char *name) {
    struct passwd *ptr;
    
    setpwent();
    while((ptr = getpwent()) != NULL) {
        if(strcmp(name, ptr->pw_name) == 0)
            break;
    }
    endpwent();
    return ptr;
}
```

# 阴影口令

加密口令是经单向加密算法处理过得用户口令副本。

因为算法是单向的，所以不能从加密口令猜出原来的口令。但是可以通过猜测原来的口令然后经过算法加密，用最终得到的口令与加密口令进行对比。所以加密口令也需要隐藏起来。

阴影口令文件一般是`/etc/shadow/`，`<shadow.h>`中定义了`spwd`结构来记阴影口令文件中的一项数据，结构如下

```c
#include <shadow.h>

struct spwd {
    char *sp_namp;		// 用户登录名
    char *sp_pwdp;		// 加密口令
    int sp_lstchg;		// 上次更改口令以来经过的时间
    int sp_min;			// 经多少天后允许更改
    int sp_max;			// 要求更改尚余天数
    int sp_warn;		// 超期警告天数
    int sp_inact;		// 账户不活动之前尚余天数
    int sp_expire;		// 账户超期天数
    unsigned int sp_flag;//保留
};
```

* 用户登录名和加密口令这两个字段是必须的，其他的字段控制口令的更改频率。
* 阴影口令文件不应该是一般用户可以读取的，有了阴影口令文件后，普通口令文件可以由用户自由读取

## 操作阴影口令文件的函数

```c
#include <shadow.h>

struct spwd *getspnam(const char *name);
struct spwd *getspent(void);
// 成功返回指针，出错返回NULL
void setspent(void);
void endspent(void);
```

这些函数只能由有`root`权限的进程调用

**FreeBSD(8.0)和MacOS(10.6.8)中没有阴影口令接口**

# 组文件

组文件一般指的是：`/etc/group`

`<grp.h>`中包含了`group`结构，定义如下：

```c
#include <grp.h>

struct group {
    char *gr_name;		// 组名
    char *gr_passwd;	// 加密口令
    int gr_gid;			// 数值组ID
    char **gr_mem;		// 指向各用户名指针的数组
};
```

* `gr_mem`字段是一个指针数组，每个指针指向一个属于该组的用户名。数组以`null`指针结尾

## 获取组文件项的函数

使用下面两个函数获取组结构

```c
#include <grp.h>

struct group *getgrgid(gid_t gid);
struct group *getgrnam(const char *name);
// 成功返回指针，出错返回NULL
```

和口令文件一样，这两个函数返回的也是函数内部的一个静态指针，每次调用时都会重写改静态变量

## 遍历整个组文件项的函数

```c
#include <grp.h>

struct group *getgrent(void);
// 成功返回指针，出错返回NULL

void setgrent(void);
void endgrent(void);
```

# 附属组ID

有了附属组ID的概念后，一个用户最多可以属于`NGROUPS_MAX`个组。

## 获取和设置附属组ID的函数

```c
#include <unistd.h>

int getgroups(int gidsetsize, gid_t grouplist[]);
// 成功返回附属组ID的数量，出错返回-1

#include <grp.h> //在linux中
#include <unistd.h> // 在FreeBSD、MacOS、Solaris中

int setgroups(int ngroups, const gid_t grouplist[]);
// 成功返回0，出错返回-1

#include <grp.h> // 在linux和Solaris中
#include <unistd.h> // 在FreeBSD和MacOS中

int initgroups(const char *username, gid_t basegid);
// 成功返回0，出错返回-1
```

* `getgroups`将进程所属用户的各个附属组ID填入`grouplist`中，填入的数量最多为`gidsetsize`个，实际填入的值由函数返回值表示。
  * `gidsetsize`为`0`时，返回附属组ID数，但不修改数组的值
* `setgroups`为调用进程设置附属组ID表。`ngroups`表示数组的元素数量，不能大于`NGROUPS_MAX`
* `initgroups`读整个组文件，然后对`username`确定其组的成员关系，然后，他调用`setgroups`，以便为该用户初始化附属组ID表。`baseid`是口令文件中的组ID
* 后面两个函数需要超级用户权限。

# 其他数据文件

除了前面提到的用户口令文件和阴影口令文件。Unix系统还提供了很多数据文件：

* `/etc/services`：记录各网络服务器所提供服务的数据文件，有每种服务所使用的默认端口
* `/etc/protocols`：记录协议信息的数据文件
* `/etc/networks`：记录网络信息的数据文件
* 等等等。。。。。。

一般情况下，对于这些数据文件，都提供了和口令文件相同的`API`，通常情况下至少用三种函数：

1. get函数：读取下一个记录项，一般返回一个结构指针
2. set函数：打开相应的数据文件，然后反饶文件
3. end函数：关闭相应的数据文件

另外，如果数据文件支持某种形式的键搜索，也会提供搜索具有指定键记录的程序

下面记录一下常见的数据文件，以及处理它们的`API`：

| 说明 |     数据文件     |    头文件    |    结构    |           附加的键搜索函数           |
| :--: | :--------------: | :----------: | :--------: | :----------------------------------: |
| 口令 |  `/etc/passwd`   |  `<pwd.h>`   |  `passwd`  |        `getpwnam`、`getpwuid`        |
|  组  |   `/etc/group`   |   `<grp.h`   |  `group`   |        `getgrnam`、`getgrgid`        |
| 阴影 |  `/etc/shadow`   | `<shadow.h>` |   `spwd`   |              `getspnam`              |
| 主机 |   `/etc/hosts`   | `<netdb.h>`  | `hostnet`  |     `getnameinfo`、`getaddrinfo`     |
| 网络 | `/etc/networks`  | `<netdb.h>`  |  `netent`  |    `getnetbyname`、`getnetbyaddr`    |
| 协议 | `/etc/protocols` | `<netdb.h>`  | `protoent` | `getprotobyname`、`getprotobynumber` |
| 服务 | `/etc/services`  | `<netdb.h>`  | `servent`  |   `getservbyname`、`getservbyport`   |

# 登录账户记录

大多数Unix系统提供下列两个数据文件：

* `utmp`文件记录当前登录到系统的各个用户
  * Linux中在`/var/run/utmp`
* `wtmp`文件跟踪各个登录和注销事件
  * Linux中在`/var/log/wtmp`

每次写入这两个文件中的是下面这个结构的二进制记录：

```c
struct utmp {
    char ut_line[8]; // tty line: "ttyh0", "ttyh1"...
    char ut_name[8]; // login name
    long ut_time;	// second since epoch
};
// 这个结构只是大概，并不完整，需要结合实现去查看
```

# 系统标识

使用下面的函数，来获取与主机和操作系统有关的信息

```c
#include <sys/utsname.h>

int uname(struct utsname *name);
// 成功返回非负值，出错返回-1

struct utsname {
    char sysname[ ];	// 操作系统的名字
    char nodename[ ];	// 结点名称
    char release[ ];	// 发布号
    char version[ ];	// 版本号
    char machine[ ];	// 硬件名称
};

#include <unistd.h>

int gethostname(char *name, int namelen);
// 成功返回0，出错返回-1
```

括号里的空格，每个系统可能不一样。Linux中是`_UTSNAME_SYSNAME_LENGTH`等值。

# 时间和日期

UNIX系统内核提供的时间是自协调世界时（UTC，公元1970年1月1日 00:00:00）以来经过的秒数。我们用数据类型time_t来表示。我们称它为日历时间，它包括时间和日期。

Unix在时间方面有其他系统的区别在于：

1. 以协调统一时间而非本地时间及时
2. 可以自动进行转换，比如转换到夏令时
3. 将时间和日期作为一个量值保存

## 返回时间的函数

```c
#include <time.h>

time_t time(time_t *calptr);
// 成功返回时间值，出错返回-1
```

如果参数非空的话，那么时间值也会保存在参数指向的单元内

## 特定时钟时间

POSXI.1的实时扩展增加了对多个系统时钟的支持。时钟通过`clockid_t`类型进行标识。标准值如下

|           标识符           |           选项           |           说明           |
| :------------------------: | :----------------------: | :----------------------: |
|      `CLOCK_REALTIME`      |                          |       实时系统时间       |
|     `CLOCK_MONOTONIC`      | `_POSIX_MONOTONIC_CLOCK` | 不带负跳数的实时系统时间 |
| `CLOCK_PROCESS_CPUTIME_ID` |     `_POSIX_CPUTIME`     |    调用进程的CPU时间     |
| `CLOCK_THREAD_CPUTIME_ID`  | `_POSIX_THREAD_CPUTIME`  |    调用线程的CPU时间     |

## 获取指定时钟的函数

```c
#include <sys/time.h>

int clock_gettime(clockid_t clock_id, struct timespec *tsp);
// 成功返回0，出错返回-1
```

`clock_id`就是上表中标识符那一列的内容。

如果`clock_id = CLOCK_REALTIME`，那么它和`time`函数的作用相同，不过在系统支持搞精度时间值的情况下，返回的精度可能比`time`函数要高

```c
#include <sys/time.h>
int clock_getres(clockid_t clock_id, struct timespec *tsp);
// 成功返回0，出错返回-1
```

`clock_getres`函数把参数`tsp`指向的结构初始化为与`clock_id`参数对应的时钟精度。比如如果精度为1毫秒，那么`tv_sec`字段为0，`tv_nsec`字段为1000000

## 设置指定时钟的函数

```c
#include <sys/time.h>
int clock_settime(clockid_t clock_id, const timespec *tsp);
// 成功返回0，出错返回-1
```

这个函数需要一定的权限才可以使用，而且有些时钟是不能修改的。

## 时间转换

通过上面的函数获取到时间值后，通常都需要调用其他函数将值转换成分解的时间结构，方便人们读取。

### 将日历时间转换成分解的时间

```c
struct tm {
    int tm_sec;		// [0 - 60]
    int tm_min;		// [0 - 59]
    int tm_hour;	// [0 - 23]
    int tm_mday;	// [1 - 31]
    int tm_mon;		// [0 - 11]
    int tm_year;	// 1900年以后的值
    int tm_wday;	// [0 - 6]
    int tm_yday;	// [0 - 365]
    int tm_isdst;	// 夏令时 <0, 0, >0
};
```

* 秒可以超过59是因为可以表示润秒
* 夏令时生效则为正，不可用为负，非夏令时为0

`time_t`转换成`struct tm`的函数

```C
#include <time.h>
struct tm *gmtime(const time_t *calptr);
struct tm *localtime(const time_T *calptr);
// 成功返回指针，出错返回NULL
```

`struct tm`转换成`time_t`的函数

```c
#include <time.h>
time_t mktime(struct tm *tmptr);
// 成功返回日历时间，失败返回-1
```

## 格式化时间的函数

将分解的时间`tm`像`printf`函数一样格式化成字符串的函数。这个函数相当复杂，主要是参数太多了。

```c
#include <time.h>
size_t strftime(char *restrict buf, size_t maxsize, 
               const char * restrict format,
               const struct tm *restrict tmptr);
size_t strftime_l(char *restrict buf, size_t maxsize,
                 const char *restrict format,
                 const struct tm *restrict tmptr,
                 local_t locale);
// 成功返回存入数组的字符数，出错返回0
```

`strftime_l`允许调用者将区域指定为参数，`strftime`使用环境变量`TZ`指定。

### 参数说明

`buf`和`maxsize`指定了存放最终格式化字符串的空间和空间大小

`format`控制时间值的格式。下面是37中`ISo C`规定的转换说明：

| 格式 |                  说明                  |           实例           |
| :--: | :------------------------------------: | :----------------------: |
| `%a` |              缩写的周日名              |           Thu            |
| `%A` |              全写的周日名              |         Thursday         |
| `%b` |               缩写的月名               |           Jan            |
| `%B` |              全写写的月名              |         Jannuary         |
| `%c` |               日期和时间               | Thu Jan 19 21:24:52 2012 |
| `%C` |             年/100(00~99)              |            20            |
| `%d` |             月日（01~31）              |            19            |
| `%D` |            日期（MM/DD/YY）            |         01/19/12         |
| `%e` |    月日（一位数字前面加空格 1~31）     |            19            |
| `%F` |     ISO 8601 日期格式（YYYY-MM-DD)     |                          |
| `%g` | ISO 8601 基于周的年的最后两位（00~99） |                          |
| `%G` |          ISO 8601 基于周的年           |                          |
| `%h` |                与%b相同                |                          |
| `%H` |       小时（24小时制）（00~23）        |                          |
| `%I` |       小时（12小时制）（01~12）        |                          |
| `%j` |            年日（001~366）             |                          |
| `%m` |              月（01~12）               |                          |
| `%M` |              分（00~59）               |                          |
| `%n` |                 换行符                 |                          |
| `%p` |                 AM/PM                  |                          |
| `%r` |           本地时间(12小时制)           |       09:24:52 PM        |
| `%R` |             与"%H:%M"相同              |          21:24           |
| `%S` |              秒（00~60）               |                          |
| `%t` |               水平制表符               |                          |
| `%T` |            与"%H:%M:%S"相同            |         21:24:52         |
| `%u` |          ISO 8601 周几（1~7）          |            4             |
| `%U` |          星期日周数（00~53）           |            03            |
| `%V` |             ISO 8601 周数              |            4             |
| `%w` |              周几（0~6）               |            4             |
| `%W` |          星期一周数（00~53）           |            03            |
| `%x` |                本地日期                |         01/19/12         |
| `%X` |                本地时间                |         21:24:52         |
| `%y` |          年的后两位(00 ~ 99)           |            12            |
| `%Y` |                   年                   |           2012           |
| `%z` |        ISO 8601格式的UTC偏移量         |          -0500           |
| `%z` |                 时区名                 |           EST            |
| `%%` |                翻译为%                 |            %             |

## 字符串转时间的函数

```c
#include <time.h>
char *strptime(const char *restrict buf, const char *restrict format, struct tm *restrict tmptr);
// 成功返回指向上次解析的字符的下一个字符的指针，出错返回NULL
```

## 时间小结

`localtime`、`mktime`和`strftime`受环境变量`TZ`的影响。如果定义了`TZ`，则这些函数使用其值代替系统默认时区，如果`TZ`为空，则使用同一时间`UTC`。
