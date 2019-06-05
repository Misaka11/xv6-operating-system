# xv6调试

## 1.启动调试环境
假如你写好了一个xv6操作系统，放在~/xv6revise目录下。那么，首先在命令行下不带任何参数执行cd进入你默认的用户目录。在此目录下新建一个.gdbinit文件，里面就一行字 add-auto-load-safe-path ~/xv6revise/.gdbinit 这句话的意思是，将xv6revise目录下的.gdbinit加入safe path。如果你没做这件事件，在做后面步骤时，命令行会出错，提示你这样做。

之后，在命令行通过cd命令进入你的xv6revise目录，执行
~~~
$ make qemu-gdb
~~~
此时，qemu已经启动，你的命令行窗口被挂起，这是因为在等待客户端发命令了。

此时打开另外一个命令行窗口ssh登陆，进入到xv6revise目录下，输入 $gdb 如果你看到了类似如下的提示，说明你成功了
~~~
The target architecture is assumed to be i8086
[f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file kernel
(gdb)
~~~
其中最后一行的(gdb)是命令行提示符，你可以在后面输入命令。你将可以在这个命令行里操控第一个终端挂起的xv6了。

## 2.调试xv6

1. 在main函数出打断点，确定内核空间的栈地址，可得内核空间的栈地址是0x8010b000~0x8010bfff
~~~
(gdb) b main
Breakpoint 1 at 0x80102e60: file main.c, line 19.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x80102e60 <main>:   lea    0x4(%esp),%ecx

Breakpoint 1, main () at main.c:19
19      {
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
2. 在scheduler()打断点，运行到断点并单步执行一步，查看寄存器的值
~~~
(gdb) b scheduler
Breakpoint 1 at 0x80103a40: file proc.c, line 324.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x80103a40 <scheduler>:      push   %ebp

Breakpoint 1, scheduler () at proc.c:324
324     {
(gdb) si
=> 0x80103a41 <scheduler+1>:    mov    %esp,%ebp
0x80103a41      324     {
(gdb) gdb reg
Undefined command: "gdb".  Try "help".
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

%cs=8，后两位为00表示内核态，%esp=0x8010b578,是内核空间的栈地址。

3. shutdown函数在系统调用前的栈的值为
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
%cs=27，后两位为11，在用户态，%esp=0x2fc0,是进程的用户栈。

4. shutdown在进入系统调用后的状态
~~~
(gdb)
=> 0x337:       int    $0x40
0x00000337 in ?? ()
(gdb)
=> 0x80105d49 <vector64+2>:     push   $0x40
vector64 () at vectors.S:320
320       pushl $64
(gdb) info reg
eax            0x16     22
ecx            0x710    1808
edx            0x0      0
ebx            0xbfa8   49064
esp            0x8dfbefe8       0x8dfbefe8
ebp            0x2fd8   0x2fd8
esi            0x0      0
edi            0x0      0
eip            0x80105d49       0x80105d49 <vector64+2>
eflags         0x212    [ AF IF ]
cs             0x8      8
ss             0x10     16
ds             0x23     35
es             0x23     35
fs             0x0      0
gs             0x0      0
~~~

5. outw函数调试过程
~~~
(gdb) b outw
Breakpoint 2 at 0x80105638: file x86.h, line 30.
(gdb) c
Continuing.
=> 0x80105638 <sys_shutdown+40>:        mov    $0x604,%edx

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
(gdb) si
=> 0x8010563d <sys_shutdown+45>:        mov    $0x2000,%eax
0x8010563d in outw (data=8192, port=1540) at x86.h:30
30        asm volatile("out %0,%1" : : "a" (data), "d" (port));
(gdb) info reg
eax            0x200    512
ecx            0x1      1
edx            0x604    1540
ebx            0x80112e4c       -2146357684
esp            0x8dfbef40       0x8dfbef40
ebp            0x8dfbef68       0x8dfbef68
esi            0x8dfbefb4       -1912868940
edi            0x8dfbefb4       -1912868940
eip            0x8010563d       0x8010563d <sys_shutdown+45>
eflags         0x282    [ SF IF ]
cs             0x8      8
ss             0x10     16
ds             0x10     16
es             0x10     16
fs             0x0      0
gs             0x0      0

~~~
%cs=8，后两位为00，内核态。%esp=0x8dfbef40，不是内核空间的栈，而是用户的内核栈

6. swtch函数调试过程
~~~
(gdb) b swtch
Breakpoint 1 at 0x8010466b: file swtch.S, line 11.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x8010466b <swtch>:  mov    0x4(%esp),%eax

Breakpoint 1, swtch () at swtch.S:11
11        movl 4(%esp), %eax
(gdb) bt
#0  swtch () at swtch.S:11
#1  0x80103ab5 in scheduler () at proc.c:346
#2  0x80102e3f in mpmain () at main.c:57
#3  0x80102f7f in main () at main.c:37
(gdb) si
=> 0x8010466f <swtch+4>:        mov    0x8(%esp),%edx
12        movl 8(%esp), %edx
(gdb) bt
#0  swtch () at swtch.S:12
#1  0x80103ab5 in scheduler () at proc.c:346
#2  0x80102e3f in mpmain () at main.c:57
#3  0x80102f7f in main () at main.c:37
(gdb) si
=> 0x80104673 <swtch+8>:        push   %ebp
15        pushl %ebp
(gdb) bt
#0  swtch () at swtch.S:15
#1  0x80103ab5 in scheduler () at proc.c:346
#2  0x80102e3f in mpmain () at main.c:57
#3  0x80102f7f in main () at main.c:37
(gdb) si
=> 0x80104674 <swtch+9>:        push   %ebx
swtch () at swtch.S:16
16        pushl %ebx
(gdb) bt
#0  swtch () at swtch.S:16
#1  0x8010b578 in stack ()
#2  0x80103ab5 in scheduler () at proc.c:346
#3  0x80102e3f in mpmain () at main.c:57
#4  0x80102f7f in main () at main.c:37
(gdb) si
=> 0x80104675 <swtch+10>:       push   %esi
swtch () at swtch.S:17
17        pushl %esi
(gdb) bt
#0  swtch () at swtch.S:17
#1  0x80112dd0 in ptable ()
#2  0x8010b578 in stack ()
#3  0x80103ab5 in scheduler () at proc.c:346
#4  0x80102e3f in mpmain () at main.c:57
#5  0x80102f7f in main () at main.c:37
(gdb) si
=> 0x80104676 <swtch+11>:       push   %edi
swtch () at swtch.S:18
18        pushl %edi
(gdb) bt
#0  swtch () at swtch.S:18
#1  0x80112780 in ?? ()
#2  0x80102e3f in mpmain () at main.c:57
#3  0x80102f7f in main () at main.c:37
(gdb) si
=> 0x80104677 <swtch+12>:       mov    %esp,(%eax)
swtch () at swtch.S:21
21        movl %esp, (%eax)
(gdb) bt
#0  swtch () at swtch.S:21
#1  0x80112784 in cpus ()
#2  0x80112780 in ?? ()
#3  0x80102e3f in mpmain () at main.c:57
#4  0x80102f7f in main () at main.c:37
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
(gdb) si
=> 0x8010467c <swtch+17>:       pop    %esi
swtch () at swtch.S:26
26        popl %esi
(gdb) bt
#0  swtch () at swtch.S:26
#1  0x00000000 in ?? ()
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
(gdb) si
=> 0x8010467d <swtch+18>:       pop    %ebx
swtch () at swtch.S:27
27        popl %ebx
(gdb) bt
#0  swtch () at swtch.S:27
#1  0x00000000 in ?? ()
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
(gdb) si
=> 0x8010467e <swtch+19>:       pop    %ebp
swtch () at swtch.S:28
28        popl %ebp
(gdb) bt
#0  swtch () at swtch.S:28
#1  0x00000000 in ?? ()
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
(gdb) si
=> 0x8010467f <swtch+20>:       ret
swtch () at swtch.S:29
29        ret
(gdb) bt
#0  swtch () at swtch.S:29
#1  0x80103670 in allocproc () at proc.c:97
~~~

## 3.调试遇到问题及解决

Q：make qemu-gdb进入QEMU界面，然后通过关闭终端退出，再次make qemu-gdb时报错：“qemu-system-i386: -gdb tcp::25000: Failed to bind socket: Address already in use”，怎么解决？

A: 发生这种问题是由于端口被程序绑定而没有释放造成。可以使用netstat -lp命令查询当前处于连接的程序以及对应的进程信息。然后用ps pid察看对应的进程，并使用kill pid关闭该进程即可。
