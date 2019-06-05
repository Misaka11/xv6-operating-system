proj0实验
===

# 1.编译、运行xv6
按照教程拷贝一份文件并运行，正确运行


# 2.添加用户命令
## 实验过程
* 分析可知print31命令是用户命令，用于接收字符串并自定义显示。与echo命令很像
* 找到echo.c文件，仿照其写了一个print31.c文件。
~~~
#include "types.h"
#include "stat.h"
#include "user.h"

int
main(int argc, char *argv[])
{
  int i;
    printf(1,"OS Lab 161630131: ");
  for(i = 1; i < argc; i++)
    printf(1, "%s%s", argv[i], i+1 < argc ? " " : "\n");
  exit();
}
~~~
* 观看源码可知，添加一条用户命令只需要在Makefile相应添加一行 _print31\
* 编译，调试，运行成功.
~~~
print31 hello,world
OS Lab 161630131: hello,world
~~~
## 文件作用
* print31.c 定义指令的具体行为。接收字符串并自定义显示
* Makefile 负责编译文件。在其中添加新增的用户命令。

# 3.添加内核输出语句
在main.c中，在cprintf("cpu%d: starting %d\n", cpuid(), cpuid());下添加一行打印信息，打印自己的姓名学号。cprintf("yize 161630131\n");
## 内核输出和用户空间输出的不同
* 调用方式不同：内核通过cprintf函数输出，用户通过printf函数输出
* 函数类型不同： cprintf属于在系统内核定义的函数，而printf属于用户空间的函数，它通过系统调用write函数完成输出。
* 输出目的不同：cprintf向控制台输出，printf向标准输出流输出。

# 4.添加系统调用
## 实验过程
* 添加文件shutdown.c，在其中使用系统调用shutdown
* 在syscall.c添加系统调用shutdown的声明
~~~
extern int sys_shutdown(void);
...
[SYS_shutdown] sys_shutdown,
~~~
* 在syscall.h添加shutdown的枚举值
~~~
#define SYS_shutdown 22
~~~
* 在sysproc.c中添加shutdown的定义.
~~~
//shutdown
void
sys_shutdown(void)
{
  outw(0x604,0x2000);
}
~~~
## 文件作用
* shutdown.c：用户命令，使用系统调用void shutdown(void)。
* syscall.c: 存放系统调用的声明
* syscall.h: 存放系统调用的枚举值
* sysproc.c: 存放系统调用的具体调用
* 还有一个文件sysfile.c，存放着关于文件的系统调用，此次实验没有更改。
