### 核心思路

改进前，内核获取从用户空间传入的虚拟地址，然后再copyin/copyout时，需要先查找用户页表，找到虚拟地址对应的物理内存，然后，再根据得到的物理内存去查找内核页表，从而再内核态下操作内存，需要两次查找页表的操作。

改进的目的是希望在内核态下直接通过用户空间下的虚拟地址直接查找内核页表从而找到对应的物理地址，只查找一次页表。每个用户空间都不一样，所以需要为每一个用户进程创建自己的内核页表。

### 步骤总结

添加新的内核页表，与用户页表一起创建，删除，并且在schedule函数中切换内核页表

1. 修改struct proc结构体，添加新的字段，用于保存新的内核页表

2. 参考原代码中全局内核页表的创建函数kvmmap（）和kvminit()函数，然后，添加自己的方法初始化自己创建的内核页表，在自己的内核页表中建立物理地址和虚拟地址的映射关系，然后初始化自己的内核页表，初始化的内容与全局内核页表一样

3. 自己创建的内核页表应该随每个进程的用户页表一起创建，也就是在allocproc（）函数中创建，创建时，只需要初始化即可。

4. 为每个进程分配一个内核栈，并将内核栈映射到内核页表中。

5. 进程切换时，是在内核态中，这时候应该使用用户进程创建的对应的内核页表，所以，进程实际切换前，要更改stap寄存器和页表缓存，让他们指向进程对应的内核页表。

6. 进程结束时，应该要释放内核页表，他应该和用户页表一起释放，所以在freeproc（）函数中释放。先释放内核栈，然后释放页表

7. 将用户页表的内容映射到内核页表中，目前，已经有了一个初始化的内核页表和用户页表。在内核更改进程的用户映射的每一处 （`fork()`, `exec()`, 和`sbrk()`）,都复制一份到进程的内核页表。

8. 替换掉原有的`copyin()`和`copyinstr()`，让需要解引用用户页表时，使用内核页表。
   
   8.1 在copyin（）和`copyinstr()中，调用copyin_new()和`copyinstr_new()

### 修改内容

1. 添加新的内核页表，与用户页表一起创建，删除，并且在schedule函数中切换内核页表
   
   1. 修改struct proc结构体，添加新的字段，用于保存新的内核页表
      
      ```
      pagetable_t kernelpt;      // 进程的内核页表
      ```

2. 参考原代码中全局内核页表的创建函数kvmmap（）和kvminit()函数
   
   ```
   //将物理地址映射到内核页表中
   void kvmmap(uint64 va, uint64 pa, uint64 sz, int perm)
   {
    if(mappages(kernel_pagetable, va, sz, pa, perm) != 0)
    panic("kvmmap");
   }
   
   //初始化内核页表，
   void kvminit()
   {
     kernel_pagetable = (pagetable_t) kalloc();
     memset(kernel_pagetable, 0, PGSIZE);
   
     // uart registers
     kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);
   
     // virtio mmio disk interface
     kvmmap(VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
   
     // CLINT
     kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);
   
     // PLIC
     kvmmap(PLIC, PLIC, 0x400000, PTE_R | PTE_W);
   
     // map kernel text executable and read-only.
     kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
   
     // map kernel data and the physical RAM we'll make use of.
     kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
   
     // map the trampoline for trap entry/exit to
     // the highest virtual address in the kernel.
     kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
   }
   ```
   
   然后，添加自己的方法初始化自己创建的内核页表
   
   ```
   //在自己的内核页表中建立物理地址和虚拟地址的映射关系
   
   void 
   uvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
   {
    if(mappages(pagetable, va, sz, pa, perm) != 0)
   panic("uvmmap");
   }
   ```

```
    //初始化自己的内核页表，初始化的内容与全局内核页表一样
    pagetable_t
    proc_kpt_init(){
     pagetable_t kernelpt = uvmcreate();
     if (kernelpt == 0) return 0;
     uvmmap(kernelpt, UART0, UART0, PGSIZE, PTE_R | PTE_W);
     uvmmap(kernelpt, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
     uvmmap(kernelpt, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
     uvmmap(kernelpt, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
     uvmmap(kernelpt, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
     uvmmap(kernelpt, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
     uvmmap(kernelpt, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
     return kernelpt;
    }

3. 自己创建的内核页表应该随每个进程的用户页表一起创建，也就是在allocproc（）函数中创建，创建时，只需要初始化即可。
```

   在allocproc（）函数中的对应位置添加如下代码
   // Init the kernal page table
   p->kernelpt = proc_kpt_init();
   if(p->kernelpt == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
   }

```
4. 为每个进程分配一个内核栈，并将内核栈映射到内核页表中。

5. 进程切换时，实在内核态中，这时候应该使用用户进程对应的行创建的内核页表，所以，进程实际切换前，要更改stap寄存器和页表缓存，让他们指向进程对应的内核页表。
```

     //调用下面两个函数实现
     w_satp(MAKE_SATP(kpt));
     sfence_vma();

```
在scheduler（）调用完swtch(&c->context, &p->context);函数后，当前cpu又运行在了scheduler（）线程上，此时应该使用全局内核页表。

6. 进程结束时，应该要释放内核页表，他应该和用户页表一起释放，所以在freeproc（）函数中释放。先释放内核栈，然后释放页表

释放页表时，参考walk函数的逻辑，先根据pte找到对应的物理内存，如果pte不是最后一层，则递归调用释放函数。
```

   void
   proc_freekernelpt(pagetable_t kernelpt)
   {
     // similar to the freewalk method
     // there are 2^9 = 512 PTEs in a page table.
     for(int i = 0; i < 512; i++){
       pte_t pte = kernelpt[i];
       if(pte & PTE_V){
         kernelpt[i] = 0;
         if ((pte & (PTE_R|PTE_W|PTE_X)) == 0){
           uint64 child = PTE2PA(pte);
           proc_freekernelpt((pagetable_t)child);
         }
       }
     }
     kfree((void*)kernelpt);
   }

```
7. 将用户页表的内容映射到内核页表中，目前，已经有了一个初始化的内核页表和用户页表。

1. 用户页表初始化过程；

（1）在allocproc函数中调用  p->pagetable = proc_pagetable(p);

（2）在proc_pagetable函数中，调用uvmcreate()函数创建一个**空的页表**，uvmcreate()：分配一个内存页面，然后置零，然后在proc_pagetable函数中调用mappages将TRAMPOLINE和trapframe页面映射到页表中。

最终，得到一个已经映射了TRAMPOLINE和trapframe页面的页表

2. 在内核更改进程的用户映射的每一处 （`fork()`, `exec()`, 和`sbrk()`）,都复制一份到进程的内核页表。

   2.1 fork()：所以，子进程中，要复制父进程的内核页表

       2.1.1 调用allocproc（）函数，allocproc（）函数申请一个可用的进程结构体，然后分配一个**pid**，创建一个**只含有TRAMPOLINE和trapframe页面映射**的**用户页表**，然后设置p->context的内容

       2.1.2  然后复制父进程的各种状态，包括页表，size，trapframe，打开文件表，状态等

   2.2 sbrk（）：比如，调用malloc时，进程就需要更多的内存

   因为新分配了物理页面并映射到页表中，所以页表也发生了变换，所以这里将所有的页表项，复制到内核页表中。通过walk去寻找pte，哪怕已经存在，也需要重新在内核页表中设置一遍

   2.3 exec()：在fork之后，调用exec，会释放原来的内存，然后从文件系统中读取文件，分配新的内存，页表也发生了更改。

3. 页表复制逻辑：利用walk从用户页表中读取pte，同样的用walk从内核页表中读取pte，如果读不到，则分配一个页面，然后，利用最低一级页表读取的pte去设置内核页表的pte，从而实现复制。
```
