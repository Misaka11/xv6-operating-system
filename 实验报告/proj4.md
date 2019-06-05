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