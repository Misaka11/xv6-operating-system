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