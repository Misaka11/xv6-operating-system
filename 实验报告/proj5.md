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