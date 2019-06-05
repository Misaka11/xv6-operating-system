# 操作系统实验总报告

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


proj1实验
===

# 1.带参数的系统调用
## 实验过程
* 阅读sleep函数实现，发现内核通过argint获取用户传递参数，参数为int型
~~~
// Fetch the nth 32-bit system call argument.
int
argint(int n, int *ip)
{
  return fetchint((myproc()->tf->esp) + 4 + 4*n, ip);
}
~~~
> *  第一个参数n，即获取第n个参数的值。第二个参数ip是获取用户传递参数的返回指针。
> * 4\*n的4是指一个int占四个字节。为什么是esp+4，而不是esp。原因是esp用于指向栈的栈顶（下一个压入栈的活动记录的顶部），而栈由高地址向低地址成长，函数调用是用入栈的方式传递参数，故在函数处理参数时，esp+4就是最后一个入栈的参数的地址。esp+4+4\*n就是第n个参数的值了。
* 修改sysproc.c中系统调用shutdown的定义。获取输入参数，打印语句。
~~~
int
sys_shutdown(void){
  int n;
  if(argint(0,&n)<0)
    return -1;
  cprintf("Leaving with code %d.\n",n);
  outw(0x604, 0x2000);
  return 0;
}
~~~
* 修改shutdown.c。增加获取int输入参数语句。
~~~
int
main(int argc, char *argv[])
{
  int a = atoi(argv[1]);
  shutdown(a);
  exit();
}
~~~

# 2.理解进程切换

* CPU在执行scheduler()时运行在用户态还是内核态？运行在哪个栈上面？

在main函数的入口处打断点，查看寄存器，可得内核空间的栈地址是0x8010b000~0x8010bfff
~~~
(gdb) info reg
eax            0x80102e60       -2146423200
ecx            0x0      0
edx            0x1f0    496
ebx            0x10074  65652
esp            0x8010b5d0       0x8010b5d0
ebp            0x7bf8   0x7bf8
esi            0x10074  65652
edi            0x0      0
eip            0x80102e60       0x80102e60 <main>
eflags         0x86     [ PF SF ]
cs             0x8      8
ss             0x10     16
~~~
在scheduler()函数打断点，查看寄存器
~~~
(gdb) info reg
eax            0x0      0
ecx            0x1      1
edx            0x80112780       -2146359424
ebx            0x0      0
esp            0x8010b578       0x8010b578 <stack+4024>
ebp            0x8010b598       0x8010b598 <stack+4056>
esi            0x10074  65652
edi            0x0      0
eip            0x80103a41       0x80103a41 <scheduler+1>
eflags         0x82     [ SF ]
cs             0x8      8
ss             0x10     16
ds             0x10     16
es             0x10     16
fs             0x0      0
gs             0x0      0
~~~
> %cs=8，后两位为00表示内核态，%esp=0x8010b578,是内核空间的栈地址。

* 当你在命令行敲下shutdown时，系统会创建一个进程执行shutdown.c中的代码，该代码中调用了系统调用void shutdown()，那么，在该进程的执行过程中，在main函数之内、系统调用之前，CPU运行在用户态还是内核态？运行在哪个栈上面？
~~~
(gdb) info reg
eax            0x5b     91
ecx            0x710    1808
edx            0x0      0
ebx            0xbfa8   49064
esp            0x2fc0   0x2fc0
ebp            0x2fd8   0x2fd8
esi            0x0      0
edi            0x0      0
eip            0x3f     0x3f
eflags         0x212    [ AF IF ]
cs             0x1b     27
ss             0x23     35
ds             0x23     35
es             0x23     35
fs             0x0      0
gs             0x0      0
~~~
> %cs=27，后两位为11，在用户态，%esp=0x2fc0,是进程的用户栈。

* 在执行命令shutdown的过程中，当cpu执行到涉及特权指令的函数outw()时，CPU运行在用户态还是内核态？运行在哪个栈上面？
~~~
Breakpoint 2, sys_shutdown () at sysproc.c:99
99        outw(0x604, 0x2000);
(gdb) info reg
eax            0x200    512
ecx            0x1      1
edx            0x0      0
ebx            0x80112e4c       -2146357684
esp            0x8dfbef40       0x8dfbef40
ebp            0x8dfbef68       0x8dfbef68
esi            0x8dfbefb4       -1912868940
edi            0x8dfbefb4       -1912868940
eip            0x80105638       0x80105638 <sys_shutdown+40>
eflags         0x282    [ SF IF ]
cs             0x8      8
ss             0x10     16
ds             0x10     16
es             0x10     16
fs             0x0      0
gs             0x0      0
~~~
> %cs=8，后两位为00，内核态。%esp=0x8dfbef40，不是内核空间的栈，而是用户的内核栈

* 为何在执行完swtch函数后，cpu没有像普通函数调用一样返回到scheduler函数中？
~~~
(gdb) si
=> 0x80104679 <swtch+14>:       mov    %edx,%esp
22        movl %edx, %esp
(gdb) bt
#0  swtch () at swtch.S:22
#1  0x80112784 in cpus ()
#2  0x80112780 in ?? ()
#3  0x80102e3f in mpmain () at main.c:57
#4  0x80102f7f in main () at main.c:37
(gdb) si
=> 0x8010467b <swtch+16>:       pop    %edi
swtch () at swtch.S:25
25        popl %edi
(gdb) bt
#0  swtch () at swtch.S:25
#1  0x00000000 in ?? ()
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
~~~
>在执行完 movl %edx, %esp 指令后，调用栈发生巨大变化，导致无法回去了。
~~~
       ————————————————————
      |  Context *new      |              高
edx->  ————————————————————               |
      |  Context *old      |              |
eax->  ————————————————————               V
      |       eip          |              低
       ————————————————————
      |       epb          |
       ————————————————————
      |       ebx          |
       ————————————————————
      |       esi          |
       ————————————————————
      |       edi          |
esp->  —————————————————————
~~~
>如图所示，其中栈中每个存储区域都是4个字节。最初esp指向eip的位置。这里需要提一下，在执行指令call时，当前指令的下一条指令的地址将会push到栈中，也就是上面看到的eip，这样就可以保证在函数返回时，可以回到函数调用的下一条指令的地方。然后swtch开始执行。通过几条push指令将当前进程的上下文压入栈中。上图表示此时的状态。
然后，通过两条指令进行进程切换 
~~~
  # Switch stacks
  movl %esp, (%eax)
  movl %edx, %esp
~~~
>eax中存储的是指向old_proc->contex的指针，old_proc->contex是指向当前进程的contex的指针，所以movl %esp, (%eax)，相当于：old_proc->contex = esp;

>执行 movl %edx, %esp， 将当前进程的栈指针指向新的进程的上下文。随着几条popl语句，寄存器存入新进程的上下文。指针eip也发生了变化，所以回不去了。


# 3.子进程优先的fork
## 实验过程
* 增加系统调用fork_winner(int winner); 添加全局变量_fork_winner作为算法开关
~~~
int _fork_winner = 0;

int
sys_fork_winner(void){
  int n;
  if(argint(0,&n)<0)
    return -1;
  _fork_winner = n;
  //cprintf("im in sysforkwin %d\n",_fork_winner);
  return 0;
}
~~~
* 更改proc.c中fork()函数。在函数即将结束时候，如果fork_winner算法启动，则父进程yield()，放弃本轮CPU。则子进程先执行，父进程后执行
~~~
  release(&ptable.lock);
  if(_fork_winner == 1)
    yield();
  return pid;
~~~
* 结果展示
~~~
$ forktest
Fork test
Set child as winner
Trial 0:  child!  parent!
Trial 1:  child!  parent!
Trial 2:  child!  parent!
Trial 3:  child!  parent!
Trial 4:  child!  parent!
Trial 5:  child!  parent!
Trial 6:  child!  parent!
Trial 7:  child!  parent!
Trial 8:  child!  parent!
Trial 9:  child!  parent!

Set parent as winner
Trial 0:  parent!  child!
Trial 1:  parent!  child!
Trial 2:  parent!  child!
Trial 3:  parent!  child!
Trial 4:  parent!  child!
Trial 5:  parent!  child!
Trial 6:  parent!  child!
Trial 7:  parent!  child!
Trial 8:  parent!  child!
Trial 9:  parent!  child!
~~~
## 回答问题
* 子进程在诞生后（以其状态标记为runnable为准），scheduler便可选择它运行。在第一次选中它执行时，在swtch的ret指令后，CPU执行的第一条指令是什么，在哪个函数中？这是在哪里设置的？
> 第一条指令是push %ebp，在forkret函数中，这是在allocproc函数里设置的。在swtch切换上下文后，CPU应该执行的下一条指令在eip中。而eip的值是fork()在申请分配新的进程调用allocproc()时赋值的
~~~
p->context->eip = (uint)forkret;
~~~

* 在父进程优先的情况下，偶尔会有子进程先于父进程打印到屏幕的情况出现；在子进程优先的情况下，偶尔也会有父进程先于子进程打印到屏幕的情况。解释可能的原因。
> 在父进程优先的情况下。父进程要准备输出或输出一部分时，正好父进程的时间片用完了，切换其它进程运行。这时切换到子进程，子进程进行输出，就会出现子进程先于父进程输出的情况。


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

proj3实验
===

# 1.信号量
## 设计思路
我们维护一个信号量池，可容纳100个信号量。每个信号量都有一把锁，同时信号量池也有一把锁来保护资源分配。每个信号量维护一个等待队列，等待自己的进程都放入队列睡觉。

alloc_sem：如果资源充足，分配一个未使用的信号量并初始化，信号量资源数目-1，返回信号量id

wait_sem：如果信号量合法，信号量的值value-1，如果值小于0，就将当前进程加入等待队列睡觉

signal_sem：如果信号量合法，信号量的值value+1，如果值小于等于0，就将喊醒一个等待队列的进程。

dealloc_sem：如果信号量被使用，使其变成未使用，信号量资源数目+1。同时杀死等待队列中的进程。

## 实验过程
1.定义信号量及信号量池
~~~
struct semaphore{
  char used;  // if the semaphore have been used
  int v;  // the value of semaphore
  int h,t;  // the head and tail of waitqueue
  struct proc *waitqueue[NPROC+1];  //process waitqueue
  struct spinlock lock; // lock to protect semaphore
};

struct{
  struct semaphore s[SEMNUM]; //the semaphores
  int sresource;  // semaphore resource 
  struct spinlock srlock; // the lock to protect allocation of sresource
}stable;
~~~
2.定义alloc_sem函数
~~~
int alloc_sem (int v){
  if(v<0) return -1;
  //test sresource is enough
  acquire(&stable.srlock);
  if(stable.sresource <= 0){
    release(&stable.srlock);
    return -1;
  }
  stable.sresource --;
  release(&stable.srlock);

  int i;
  for(i=0;i<SEMNUM;i++){
    acquire(&stable.s[i].lock);
    if(!stable.s[i].used){
      stable.s[i].used = 1;
      stable.s[i].h = 0;
      stable.s[i].t = 0;
      stable.s[i].v = v;
      release(&stable.s[i].lock);
      return i;
    }
    release(&stable.s[i].lock);
  }

  return -1;
}
~~~
3.定义wait_sem函数
~~~
int wait_sem(int i){
  if(i>=SEMNUM || i<0) return -1;

  struct semaphore *sem = &stable.s[i];
  acquire(&sem->lock);
  if(sem->used){
    sem->v--;
    if(sem->v < 0){
      sem->waitqueue[sem->t] = myproc();
      sem->t = (sem->t + 1 ) % (NPROC+1);
      sleep(myproc(),&sem->lock);
    }
    release(&sem->lock);
    return 1;
  }
  release(&stable.s[i].lock);

  return -1;
}
~~~
4.定义signal_sem函数
~~~
int signal_sem(int i){
  if(i>=SEMNUM || i<0) return -1;

  struct semaphore *sem = &stable.s[i];
  acquire(&sem->lock);
  if(sem->used){
    sem->v++;
    if(sem->v <= 0){
      wakeup(sem->waitqueue[sem->h]);
      sem->h = (sem->h + 1) % (NPROC+1);
    }
    release(&sem->lock);
    return 1;
  }
  release(&sem->lock);

  return -1;
}
~~~
5.定义dealloc_sem函数
~~~
int dealloc_sem(int i){
  if(i>=SEMNUM || i<0) return -1;
  
  acquire(&stable.s[i].lock);
  if(stable.s[i].used){
    stable.s[i].used = 0;
    acquire(&ptable.lock);
    int j;
    for(j=stable.s[i].h;j!=stable.s[i].t;j=(j+1)%(NPROC+1)){
      struct proc *p = stable.s[i].waitqueue[j];
      /*p->killed = 1;
      // Wake process from sleep if necessary.
      if(p->state == SLEEPING)
        p->state = RUNNABLE;*/
      kfree(p->kstack);
      p->kstack = 0;
      freevm(p->pgdir);
      p->pid = 0;
      p->parent = 0;
      p->name[0] = 0;
      p->killed = 0;
      p->state = UNUSED;
    }
    release(&ptable.lock);
    release(&stable.s[i].lock);

    acquire(&stable.srlock);
    stable.sresource++;
    release(&stable.srlock);
    return 1;
  }
  release(&stable.s[i].lock);

  return -1;
}
~~~

## 回答问题
>关于你实现的wait_sem，如果多个进程同时执行wait_sem，但wait的具体信号量不同，它们能并发执行吗？（你应该用一个spinlock去保护一个信号量，另外用一个spinlock保护资源分配）

能并发执行。1.每个信号量都有自己的锁，它们不会互相干扰。2.wait不涉及资源分配，不用拿资源分配的锁

# 2.消息传递
## 设计思路
我们假设现在有个邮箱。

进程通信有两种情况：1.发送者先把send的msg放进邮箱。2.接收者先把receive的msg放进邮箱。不存在同时放msg的情况，通过一个锁来实现。

1.先send。发送者先看邮箱里有没有接收自己的信，没有。写'send'信并等待。接收者过来，看到有发给自己的信，接收数据，将信标记为回收状态（不同于空状态，必须让发送者回收），喊发送者，然后接收者结束。发送者过来删除信，结束。

2.先receive。接收者先看邮箱里有没有发给自己的信。没有，写'receive'信并等待。发送者过来，看到有接收自己的信，直接改写该信为自己要发送的信。喊一下接收者并结束。接收者过来接受信的数据，并删除信，结束。

注意：信的种类，信什么时候删，谁来删。删除太早会导致信号量重复删除，删的人错了会导致重复接收同一封信。错了好几次才改对。

## 实验过程
1.定义message结构体和邮箱
~~~
struct message{
  int type; //-1:unused;0:send;1:receive;2:recycle
  int from_pid; //who send data
  int to_pid; //who receive data
  int sem_id; //semaphore id
  int da,db,dc; //data a,b,c
};

struct{
  struct message m[MSGNUM];
  struct spinlock mlock;
}mtable;
~~~
2.定义msg_send函数

首先判断一下pid是否合法
~~~
 //test pid is exist
  int flag = 0;
  acquire(&ptable.lock);
  struct proc *p;
  for(p=ptable.proc;p<&ptable.proc[NPROC];p++){
    if(p->pid == pid){
      flag = 1;
      break;
    }
  }
  release(&ptable.lock);
  if(!flag) return -1;
~~~
如果接收信先到了，改信，喊接收者，结束
~~~
  acquire(&mtable.mlock);
  // first receive
  int j;
  for(j=0;j<MSGNUM;j++){
    if(mtable.m[j].type == 1 && mtable.m[j].to_pid == pid){
      mtable.m[j].type = 0;
      mtable.m[j].from_pid = myproc()->pid;
      mtable.m[j].da = a;
      mtable.m[j].db = b;
      mtable.m[j].dc = c;
      signal_sem(mtable.m[j].sem_id);
      release(&mtable.mlock);
      return 1;
    }
  }
~~~
如果自己先来，写信并等待
~~~
  // send 'send' msg into box
  int i;
  for(i=0;i<MSGNUM;i++){
    if(mtable.m[i].type == -1){
      mtable.m[i].sem_id = alloc_sem(0);
      //system resource is not enough
      if(mtable.m[i].sem_id == -1){
        release(&mtable.mlock);
        return -1;
      }
      mtable.m[i].type = 0;
      mtable.m[i].from_pid = myproc()->pid;
      mtable.m[i].to_pid = pid;
      mtable.m[i].da = a;
      mtable.m[i].db = b;
      mtable.m[i].dc = c;
      break;
    }
  }
  release(&mtable.mlock);
  // wait; first send
  wait_sem(mtable.m[i].sem_id);
~~~

删除自己的信
~~~
  //delete my msg
  acquire(&mtable.mlock);
  mtable.m[i].type = -1;
  dealloc_sem(mtable.m[i].sem_id);
  release(&mtable.mlock);
  
  return 1;
~~~
3.定义msg_receive函数

首先，发送者已经到了，取信，标记信并喊发送者，自己结束
~~~
  // if first send
  int j;
  for(j=0;j<MSGNUM;j++){
    if(mtable.m[j].type == 0 && mtable.m[j].to_pid == myproc()->pid){
      *a = mtable.m[j].da;
      *b = mtable.m[j].db;
      *c = mtable.m[j].dc;
      from_pid = mtable.m[j].from_pid;
      signal_sem(mtable.m[j].sem_id);
      mtable.m[j].type = 2;
      release(&mtable.mlock);
      return from_pid;
    }
  }
~~~
如果自己先到，写信并等待
~~~
 // send 'receive' msg into box
  int i;
  for(i=0;i<MSGNUM;i++){
    if(mtable.m[i].type == -1){
      //system resource is not enough
      mtable.m[i].sem_id = alloc_sem(0);
      if(mtable.m[i].sem_id == -1){
        release(&mtable.mlock);
        return -1;
      }
      mtable.m[i].type = 1;
      mtable.m[i].to_pid = myproc()->pid;
      break;
    }
  }
  release(&mtable.mlock);
  // wait, first receive
  wait_sem(mtable.m[i].sem_id);
~~~
等到发送者来喊自己，取信，删信，结束
~~~
  // receive msg
  acquire(&mtable.mlock);
  *a = mtable.m[i].da;
  *b = mtable.m[i].db;
  *c = mtable.m[i].dc;
  from_pid = mtable.m[i].from_pid;
  //delete msg
  mtable.m[i].type = -1;
  dealloc_sem(mtable.m[i].sem_id);
  release(&mtable.mlock);
  return from_pid;
~~~

## 回答问题
>如果receive先执行，但send未执行，你是如何让receive阻塞的（或者说阻塞到哪个信号量上了）？如果send先执行，但receive未执行，你是如何让send阻塞的？

如果receive先执行，但send未执行，它会发送一个receive类型的msg，在msg中分配了一个新的信号量，它阻塞在该信号量上。

如果send先执行，但receive未执行，它会发送一个send类型的msg，在msg中分配了一个新的信号量，它阻塞在该信号量上。

# 实验结果
## 信号量
~~~
$ semtest
illegal input test: success!
alloc-dealloc test: success!
created semaphores with indices 0 1 2
testing semaphore 0 with initial value 1
-4--4--4--4--4--4--4--4--4--4--4--4--4--4--4--4--4--4--4--4-
leaving(4)
-5--5--5--5--5--5--5--5--5--5--5--5--5--5--5--5--5--5--5--5-
leaving(5)
-6--6--6--6--6--6--6--6--6--6--6--6--6--6--6--6--6--6--6--6-
leaving(6)
-7--7--7--7--7--7--7--7--7--7--7--7--7--7--7--7--7--7--7--7-
leaving(7)
-8--8--8--8--8--8--8--8--8--8--8--8--8--8--8--8--8--8--8--8-
leaving(8)
testing semaphore 1 with initial value 2
-9-10---9--10---10-9--10--9-10---9--10--9--10--9--10---10-9--9--10--9--10--9--10--9--10--9--10--9--10--9--10--9--10--9--10--9--9--10--9--10-
leavin
leavingg(9)
(-11-10)
-1-11-2--11-12---1-11-2--11--12--11--12--11--12--11--12--11--12--11--12--11--12---1211---11--12--11--12--11--12--11--12--11--12--11--12--11--12--11--12-
leav-12-ing(11)
-13-
leaving(12)
-13--13--13--13--13--13--13--13--13--13--13--13--13--13--13--13--13--13--13-
leaving(13)
testing semaphore 2 with initial value 3
--114-5--16--14--15--16--14--15--16--14--15--16--14--1-16-14-5---14--15--16--14--15--16--14--1-16-5--14--15--16--14--15---14-16--15--14--16--15--14--16--15---15--16-14--14--15--16--14--15--16--14--15--16--14--15--16---15--16-14---16-
l15-eaving(14)
-17--
16--17-leaving(15)
-18-
l-17-e-18-aving(16)
-17--18--17--18--17--18---18-17--17--18--17--18--17--18--17--18--17--18--17--18--1-18-7--17--18--17--18--17--18--17--18---18-17---18-17-
l-1eaving(17)
8-
leaving(18)
normal test done
testing dealloc_sem
pid=20 is going to wait on sem 0
pid=20 wait on sempid=22 is going to wait on sem 0
pid=19 is going to wait on sem 0
 0: success
pid=21 is going to wait on sem 0
parent leaving
~~~
## 消息传递
~~~
$ msgtest

child(sender,child+3):  43(23,46) 24(23,27) 25(23,28) 26(23,2728(229(29) 3,323,3(201) 3,30) 32(2) 3,3313(23,34) 32(23,35) 33(2) 3,3634) (2335(23,38) 36(23,,39) 3737(238(23,)
341) 39(23,440(23,43) 41(23,4,40) 2) 4) 42(23,45) 63(23,66) 44(23,45(23,48) 46(23,49) 47(23,50) 48(23,51) 50(23,53) 51(23,54) 52(23,55) 53(23,56) 54(5547) 49(2323,57) (23,58) 56(23,59) 57(23,6058(259(23,62) 60) 3,61) (23,6361(23,64) 6,52) ) 2(23,65) 83(23,86) 64(23,6765(23,68) 66(23,69) 68(23,71) 69(23,7270(23,73) 72(23,75) 73(23,7674(23,77)) 71(23,74) )  76(23,79) 778(23,81) 79(23,82)80(23,83) 81(23,84) 82(23,85) ) 67(23,70) 75(23,78) 7 (23,80) 103(23,106) 84(23,87) 85(23,88) 86(23,89) 87(23,90) 889(23,92) 90(23,93) 91(23,94) 92(23,95) 994(23,97) 95(23,98) 96(23,99) 97(23,100)98(23,101) 99(23,102) 100(23,103) 101(23,18(23,91) 3(23,96)  04) 102(23,105) 12104(23,107105(23,108) 106(23,109) 107(23,113)0) 108(23,111) 109(23,1(23,126)  1110(23,113) 111(23,114) 112(23,115) 1213(23,1114(23,117) 1)16) 15(23,118) 116(23,119) 117(23,120118(23,121) 119(23,122) 120(2121( ) 32122(23,125),3,124)  123) 124(231126(23,129) 127(23,130),25(23,128)  128(23,131) 129(23,132) 130(23,133) 131(23,13132(23,135) 13127) 4) 3(23,136) 134(23,13135(23,138) 136(23,139) 137(23,140138(23,141) 139(23,142) 140(23,143) 141(23,144) 142(23,145) 1437) ) (23,146)
messages received from :144 145 146 147 148 149 150 151 152 153 154 155 156 157 158 159 160 161 162 163
Hello World! Good Morning!�
~~~

proj4实验
===

# 1.堆空间的延迟页面分配

## 实验过程
1.sbrk函数。进程通过sbrk申请空间时，若空间大小<=0，还是采用原来的growproc函数，若>0，如果栈没增长过内核地址，则增长栈而并不分配空间。
~~~
// if n<0, relese memory as before, so as n=0
  addr = myproc()->sz;
  if(n <= 0){
    if(growproc(n) < 0) return -1;
    return addr;
  }
    
  // not allocate memory actually
  // but detect whether n is so big
  if(addr + n >= KERNBASE)
    return -1;
  myproc()->sz += n;
  return addr;
~~~
2.trap.c。触发缺页中断，进入pgflt_hdl()函数。
~~~
case T_PGFLT:
    pgflt_hdl();
    break;
~~~
3.pgflt_hdl函数。检测访问地址是否非法，非法则报错、杀死进程并返回，否则分配新的一页空间，通过mappages函数把新分配空间和访问的地址的页的起始地址（用PAGEDOWN函数下降到页的起始位置）关联起来。

下面是还没加写时复制fork的代码
~~~
// access address fault    
    if( va >= KERNBASE || va > PGROUNDUP(proc->sz)){
      // mark the process as killed
      cprintf("pid %d memtest: trap %d err %d on cpu %d eip 0x%x addr 0x%x--kill proc\n",
      proc->pid,proc->tf->trapno,proc->tf->err, cpuid(),proc->tf->eip,va);
      
      proc->killed = 1;
      return;
    }

    char *mem = kalloc();
      if(mem == 0){
        cprintf("allocuvm out of memory\n");
        return;
      }
      memset(mem, 0, PGSIZE);
      if(mappages(proc->pgdir, (char*)PGROUNDDOWN(va), PGSIZE, V2P(mem), PTE_W|PTE_U) < 0){
        cprintf("allocuvm out of memory (2)\n");
        kfree(mem);
        return;
      }
      return;
~~~
4. 修改copyuvm函数，我采取的方法是，当复制到某一页不存在时，continue忽略掉。然而，memtest的Test5就会remap，一直没有解决。 后来我就和第二个写时复制fork一起写了，就莫名奇妙地可以了。代码在写时复制fork部分贴出来。


# 2.写时复制的fork
## 设计思路
页表项的第9-11位可以来做标记。主要用到PTE_U：USER,PTE_P：PRESENT,PTE_W：WRITEABLE。

在物理内存分配器中添加一个记录页表引用次数的数组，当对页进行分配和释放操作时，对应的引用次数也要增加和减少。

关于逻辑判断，什么时候决定分配空间，什么时候修改页表项，写在了代码注释中。

## 实验过程
1.更改kalloc.c中关于分配页和删除页时，对应需要增加和删减引用次数。因较为分散此处不贴出了。

2.pgflt_hdl函数全部
~~~
void
pgflt_hdl(uint errorcode){

    // get the faulting virtual address from the CR2 register 
    uint va = rcr2();
    pte_t *pte;
    struct proc* proc = myproc();
    // access address fault    
    if( va >= KERNBASE || va > PGROUNDUP(proc->sz)){
      // mark the process as killed
      cprintf("pid %d memtest: trap %d err %d on cpu %d eip 0x%x addr 0x%x--kill proc\n",
      proc->pid,proc->tf->trapno,proc->tf->err, cpuid(),proc->tf->eip,va);
      
      proc->killed = 1;
      return;
    }
    // if pte is not present, allocate a new page
    // if page is present, and belong to user, and read only,  if ref>1,allocate one new page, if ref = 1,change to writeable
    // if page is not present ,allocate a new page
    if((pte = walkpgdir(proc->pgdir, (void*)va, 0)) == 0){
      char *mem = kalloc();
      if(mem == 0){
        cprintf("allocuvm out of memory\n");
        return;
      }
      memset(mem, 0, PGSIZE);
      if(mappages(proc->pgdir, (char*)PGROUNDDOWN(va), PGSIZE, V2P(mem), PTE_W|PTE_U) < 0){
        cprintf("allocuvm out of memory (2)\n");
        kfree(mem);
        return;
      }
      *pte = V2P(mem) | PTE_P | PTE_U ;
      return;
    }
    else{
      if((*pte & PTE_P) && (*pte & PTE_U) && !(*pte & PTE_W)){
        // get the physical address from the given page table entry 
        uint pa = PTE_ADDR(*pte);
        uint refCount = getRef(pa);
        char *mem;

        // Current process is the first one that tries to write to this page
        if(refCount > 1) {
            // allocate a new memory page for the process
            if((mem = kalloc()) == 0) {
              cprintf("Page fault out of memory, kill proc %s with pid %d\n", proc->name, proc->pid);
              proc->killed = 1;
              return;
            }
            memmove(mem, (char*)P2V(pa), PGSIZE);
            *pte = V2P(mem) | PTE_P | PTE_U | PTE_W;

            // ref-1
            decRef(pa);
        }
        else if(refCount == 1){
          *pte |= PTE_W;
        }
        else{
          panic("pagefault reference error\n");
        }
      }
      else if(!(*pte & PTE_P)){
        char *mem = kalloc();
        if(mem == 0){
          cprintf("allocuvm out of memory\n");
          return;
        }
        memset(mem, 0, PGSIZE);
        if(mappages(proc->pgdir, (char*)PGROUNDDOWN(va), PGSIZE, V2P(mem), PTE_W|PTE_U) < 0){
          cprintf("allocuvm out of memory (2)\n");
          kfree(mem);
          return;
        }
        *pte = V2P(mem) | PTE_P | PTE_U ;
        return;
      }
      //other error
      else{
          // mark the process as killed
        cprintf("pid %d memtest: trap %d err %d on cpu %d eip 0x%x addr 0x%x--kill proc\n",
        proc->pid,proc->tf->trapno,proc->tf->err, cpuid(),proc->tf->eip,va);
        
        proc->killed = 1;
        return;
      }
    }

    // Flush TLB
    lcr3(V2P(proc->pgdir));
}
~~~
3.copyuvm函数全部
~~~
pde_t*
copyuvm(pde_t *pgdir, uint sz)
{
pde_t *d;
  pte_t *pte;
  uint pa, i, flags;

  if((d = setupkvm()) == 0)
    return 0;
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walkpgdir(pgdir, (void *) i, 0)) == 0)
      panic("copyuvm: pte should exist");
    if(!(*pte & PTE_P)) continue;
      //panic("copyuvm: page not present");
    *pte &= ~PTE_W;                      // read only
    pa = PTE_ADDR(*pte);
    flags = PTE_FLAGS(*pte);
    if(mappages(d, (void*)i, PGSIZE, pa, flags) < 0)
      goto bad;
    incRef(pa);
  }
  lcr3(V2P(pgdir));                     // Flush TLB 
  return d;

bad:
  freevm(d);
  lcr3(V2P(pgdir));
  return 0;
}
~~~


# 3.实验结果
## 
~~~
$ memtest
number of free frames: 56732
Test 1 (sbrk without allocating memory):  success!
Test 2 (write to heap mem):  success!
Test 3 (deallocating memory):  success!
Test 4 (allocating too much mem):  success!
Test 5 (access invalid mem, two page faults):
pid 4 memtest: trap 14 err 6 on cpu 0 eip 0x239 addr 0x7000--kill proc
pid 5 memtest: trap 14 err 5 on cpu 0 eip 0x271 addr 0x1fff--kill proc
$ forktest
fork test
fork test OK
$ stresstest
created 61 child processes
pre: 56728, post: 52520
~~~

# 4.开放任务
我的post是52520。因为能力有限，加上前面两个实验内容太难了，我就没有考虑节省物理内存。

这次实验难度有点大，我参考了GitHub上一些关于Lazy Page Allocation 和 Copy-on-write fork项目。但自己实现还是出现了许多问题。比如，完成第一个出现的remap，和第二个如何区分是延迟分配引起的还是写时复制fork引起的缺页中断。另外，forktest好像就没失败过，不管怎么改代码。。。

proj5实验
===

# 1.添加Ctrl+C快捷键

## 实验过程
1.在console.c中添加对应Ctrl+C的处理。

使用consputc输出单个字符。用cprintf会panic，不知道怎么处理。

调用CtrlC函数。返回2，则模仿default的一段代码清空缓冲区，不然在Ctrl+C之后还需要手动敲回车。
~~~
case C('C'):  //add Ctrl + C kill process
      consputc('^');
      consputc('C');
      consputc('\n');
      if(CtrlC() == 2){
        input.buf[input.e++ % INPUT_BUF] = '\n';
        input.w = input.e;
        wakeup(&input.r);
      }
      break;
~~~
2.在proc.c中添加CtrlC函数

将按下组合键时被打断的那个用户进程杀死，返回1。

如果不存在那个进程，即，如果用户进程均在阻塞状态，则将正在等待命令行输入的用户进程杀死，返回2。(Hint: 如果用户进程在等待命令行输入，则它的chan指向的是input.r)

但如果等待命令行输入的是sh进程，则忽略，返回2。

~~~
// return 1 kill present process
// return 2  "sh" or kill process waiting for input
int
CtrlC(void){
  struct proc *p;
  extern uint *pointr;
  if((p = myproc()) != 0){
    p->killed = 1;
    return 1;
  }
  acquire(&ptable.lock);
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(strncmp(p->name,"sh",2) == 0){
      //p->state = RUNNABLE;
      continue;
    }
    if(p->state == SLEEPING && p->chan == pointr){
      p->state = RUNNABLE;
      p->killed = 1;
      break;
    }
  }
  release(&ptable.lock);
  return 2;
}
~~~

## 实验结果
~~~
$ deadloop
^C
$ deadloop
^C
$ infread
hello,world
hello,world
^C
$ ^C
$ ^C
$ deadloop
^C
$
~~~
## 回答问题
>阅读infread.c，它是从控制台读入一个字符，然后打印到屏幕上。但如果你在屏幕上连续输入abcdefg，屏幕没有反应，敲回车后，屏幕才会打印abcdefg。为何不是输入a后立即显示a？指出造成这个现象的原因。（提示：main函数中调用的consoleinit会将devsw[CONSOLE].read指向consoleread()，此函数在fs.c的readi中会被调用。）

infread->read->sys_read->fileread->readi->consoleread

没敲回车之前，字符存放在缓冲区中，敲击回车之后，将缓冲区的内容当作一个文件（？），使用文件读入将数据读入程序中。

# 2.添加文件系统校验

## 实验过程
在mpmain()调用scheduler()函数之前，添加函数filecheck()。

filecheck函数：

1. 分配1页内存读入superblock，获取inode个数，bitmap起始位置，inode起始位置。
2. 分配4页内存存放dinode，一次读入所有dinode。
3. 将读入的dinode的nlink清零。
4. 遍历目录，统计dinode的nlink。
5. 检查读入的dinode，如果没有引用（nlink=0）并且没有释放（type!=0），输出修复信息，清空该dinode的各个信息（继续删除操作）
6. 写回修复的dinode，释放分配的内存空间。


## 实验结果
fs5.img
~~~
cpu0: starting 0
----------------------------------
 Running fsck ...
fsck: inode 3 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 5 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 7 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 13 is allocated but is not referenced by dir! Fixing ... done
fsck completed. Fixed 4 inodes and freed 48 disk blocks.
----------------------------------
sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap start 58


----------------------------------
 Running fsck ...
fsck: no problem found.
----------------------------------
~~~
fs5i.img
~~~
cpu0: starting 0
----------------------------------
 Running fsck ...
fsck: inode 20 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 21 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 22 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 23 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 24 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 25 is allocated but is not referenced by dir! Fixing ... done
fsck: inode 26 is allocated but is not referenced by dir! Fixing ... done
fsck completed. Fixed 7 inodes and freed 7 disk blocks.
----------------------------------
sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap start 58


----------------------------------
 Running fsck ...
fsck: no problem found.
----------------------------------
~~~

## 回答问题
>为何OS尚未切换到scheduler时，读写磁盘不能用中断的方式进行？

当前没有其它进程运行，使用中断没法切换，反而会浪费时间

>写出你遍历目录的伪代码。
因为目录采用树形结构，我采用递归遍历。
~~~
procedure getReference(int i)     // 获取第i个dinode的引用次数
  checknode = 获取第i个dinode
  if checknode->type != 1 && checknode->type != 2 && checknode->type != 3 then
    return
  checknode->nlink++
  if checknode->type == 2 || checknode->type == 3 then    //不是目录文件
    return
  if checknode->type == 1 then    // 是目录文件
  begin
    data = 分配新的一页
    for(j = 0 ; j <NDIRECT ; j++)
    begin
       if checkinode->addrs[j] != 0 then  //  存在数据区
       begin
         data = checkinode数据区内容
         for(k = 2*dirsize; k<BSIZE;k+=dirsize)
         begin
           de = 读取目录dirent内容
           if de->inum!=i && de->name != '.' && de->name != '..' then    // 不返回上级目录，避免死循环
            getReference(de->inum);
         end
       end
    end
    释放data
  end
~~~

>你是使用全局变量存储inode，还是使用局部变量或者是用kalloc和kfree？

使用kalloc和kfree。首先分配一页内存空间存放superblock，然后使用4个char*指针，分配4页内存空间。每页4096B，每个inode是64B，所以一页可以有64个inode。


# 总结

通过操作系统实验，深入xv6系统源码，学习了不少操作系统重要知识点。每当将课本上的理论知识转化为代码实现出来时，会产生巨大的成就感。特别是完全靠自己思考实现的机制能够正常运转。当然，操作系统实验总体难度较大，无论是GitHub上世界各地的大佬，还是我们班的大佬，都给予了我莫大的帮助。最后，感谢操作系统朱小军老师。老师幽默有趣而不失严谨的教学方式让我获益匪浅，印象深刻。希望以后能向朱老师学习，既能保持严谨认真的科研精神，又能有乐观幽默的人生态度。