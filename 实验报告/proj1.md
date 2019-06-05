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
