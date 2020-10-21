[toc]

# main函数主体

```c
int
main(void)
{
  kinit1(end, P2V(4*1024*1024)); // phys page allocator
  kvmalloc();      // kernel page table
  mpinit();        // detect other processors
  lapicinit();     // interrupt controller
  seginit();       // segment descriptors
  picinit();       // disable pic
  ioapicinit();    // another interrupt controller
  consoleinit();   // console hardware
  uartinit();      // serial port
  pinit();         // process table
  tvinit();        // trap vectors
  binit();         // buffer cache
  fileinit();      // file table
  ideinit();       // disk 
  startothers();   // start other processors
  kinit2(P2V(4*1024*1024), P2V(PHYSTOP)); // must come after startothers()
  userinit();      // first user process
  mpmain();        // finish this processor's setup
}
```



## kinit1()

该函数在*kalloc.c*中实现。这是*kinit1*函数的主要作用是初始化自旋锁，然后关闭锁的使用，再初始化空闲空间分配器。

## kvmalloc()





# main中定义的函数/变量

## pde_t entrypgdir[]

```c
// The boot page table used in entry.S and entryother.S.
// Page directories (and page tables) must start on page boundaries,
// hence the __aligned__ attribute.
// PTE_PS in a page directory entry enables 4Mbyte pages.

__attribute__((__aligned__(PGSIZE)))
pde_t entrypgdir[NPDENTRIES] = {
  // Map VA's [0, 4MB) to PA's [0, 4MB)
  [0] = (0) | PTE_P | PTE_W | PTE_PS,
  // Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
  [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
};
```

* 该变量全称是entry page directory，页目录条目，里面存储的是页表或者页

* pde_t类型是unsigned int的别名。

* [_\_attribute\_\_()](https://www.jianshu.com/p/dda61084f9b5)函数的作用是设置函数属性，这里是设置空间分配的大小为PGSIZE，其值为4096 Bytes。

* NPDENTRIES是每页表的条目数，定义在*mmu.h*文件中，其值为1024

* 然后分别设置了下标\[0\]和下标[0x80000000 >> 22]位置的值

  * 下列这几个常量均定义在*mmu.h*文件中

  - `0` - clear all bits
  - `PTE_P` - set the present bit
  - `PTE_W` - set the read\write bit
  - `PTE_PS` - set the 4MiB page size bit

* 另，虚拟地址在*mmu.h*文件中有定义

  ```c
  // A virtual address 'la' has a three-part structure as follows:
  //
  // +--------10------+-------10-------+---------12----------+
  // | Page Directory |   Page Table   | Offset within Page  |
  // |      Index     |      Index     |                     |
  // +----------------+----------------+---------------------+
  //  \--- PDX(va) --/ \--- PTX(va) --/
  ```

  