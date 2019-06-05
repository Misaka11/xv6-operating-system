proj2实验
===

# 1.线程支持
## 实验过程
* 添加库函数

在用户命令里面调用相应的系统调用并传递参数。
需要注意的是，在xthread_create中申请用户栈，将栈地址作为参数传入。
在xthread_join中等待线程结束后，将返回的用户栈地址用free函数释放掉。

* 添加系统调用

更改内容：

1.PCB增加用户栈指针ustack、标记是否为线程的变量isthread

2.allocproc函数，设置process默认不是线程

3.exit函数，扫描process表，杀掉当前process的孩子线程

4.clone函数，参考fork函数，把新process的pgdir指向父进程。下一条指令eip指向传入的函数。新增的用户栈指向传入的栈地址。为了获取到return的值，设置一个假的返回地址，如0xffffffff，然后在trap中捕获page fault，进行特殊处理。

5.join函数，参考wait函数。等待子线程结束并为其收尸。

6.thread_exit函数，参考exit函数，关闭自己的process并唤醒通过join等待自己的线程。

7.trap.c，对假的返回地址特殊处理，调用thread_exit返回线程调用函数的返回值

* 实验结果
~~~
$ threadtest

-------------------- Test Return Value --------------------
Child thChild thread 2: count=3
Main thread: read 1: count=3
thread 1 returned 3
Main thread: thread 2 returned 2

-------------------- Test Stack Space --------------------
ptr1 - ptr2 = 16
Return value 123

-------------------- Test Thread Count --------------------
Child process created 60 threads
Parent process created 61 threads
~~~

## 回答问题
> 在你的设计中，struct proc增加了几个字节？（优先级调度部分增加的不算）

8个字节。我增加了一个记录用户栈的指针 char *ustack,和一个记录是否是线程的变量 int isthread。

# 2.优先级调度
## 实验过程
* 更改内容

1.PCB增加线程的优先级prior

2.allocproc函数，设置默认优先级为2

3.scheduler函数，每次需要调度时，总是优先级最高的进程被调度；如果有优先级更高的出现，重新从最高级开始调度；先调用同一优先级的进程，如果没有可调用的进程，则调用低一级优先级的。如果调用完最低级的进程，则优先级从最高级开始调度。

4.sched函数，增加打印进程pid语句

5.wakeup1函数，唤醒时看看是否唤醒了优先级高的进程。

6.set_priority函数，设置进程的优先级，设置时看看是否设置了优先级高的进程。

7.enable_sched_display开关函数，设置全局变量display_enable，是否输出进程pid

* 实验结果
~~~
$ schedtest
================================
Parent (pid=129, prior=1)
Child (pid=130, prior=1) created!
Child (pid=131, prior=2) created!
Child (pid=132, prior=3) created!
Child (pid=133, prior=1) created!
Child (pid=134, prior=2) created!
Child (pid=135, prior=3) created!
Child (pid=136, prior=2) created!
================================
129 - 130 - 133 - 130 - 133 - 130 - 133 - 130 - 133 - 130 - 133 - 130 - 133 - 130 - 133 - 130 - 133 - 130 - 133 - 130 - 133 - 130 - 133 - 129 - 133 - 133 - 133 - 129 - 129 -
131 - 134 - 136 - 131 - 134 - 136 - 131 - 134 - 136 - 131 - 134 - 136 - 131 - 134 - 136 - 131 - 134 - 136 - 131 - 134 - 136 - 131 - 134 - 136 - 131 - 134 - 136 - 129 - 134 -
134 - 129 - 132 - 135 - 132 - 135 - 132 - 135 - 132 - 135 - 132 - 135 - 132 - 135 - 129 - 135 -
~~~

## 回答问题
> 父进程的优先级为1，为何有时优先级低的子进程会先于它执行？

父进程处于阻塞状态，如在printf。

>父进程似乎周期性出现在打印列表中，为什么？（你需要阅读schedtest.c） 

父进程wait了自己子进程，当子进程结束时就会唤醒一次父进程，父进程就会执行，所以周期性出现。

>set_priority系统调用会否和scheduler函数发生竞争条件（race condition）？你是如何解决的？

会，在设置优先级的时候，设置的优先级较高，就会使scheduler函数重新从最高优先级开始调度。给set_priority函数上锁。