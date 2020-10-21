该文件中定义了一些内存中会用到的宏。

```c
// Memory layout

#define EXTMEM  0x100000            // Start of extended memory
#define PHYSTOP 0xE000000           // Top physical memory
#define DEVSPACE 0xFE000000         // Other devices are at high addresses

// Key addresses for address space layout (see kmap in vm.c for layout)
#define KERNBASE 0x80000000         // First kernel virtual address
#define KERNLINK (KERNBASE+EXTMEM)  // Address where kernel is linked

#define V2P(a) (((uint) (a)) - KERNBASE)
#define P2V(a) ((void *)(((char *) (a)) + KERNBASE))

#define V2P_WO(x) ((x) - KERNBASE)    // same as V2P, but without casts
#define P2V_WO(x) ((x) + KERNBASE)    // same as P2V, but without casts

```



# V2P和P2V

这些宏所做的事情是将虚址和实址做了一个映射转换。但是基于操作系统所学的知识，这种转换通常会涉及页、页表等结构，而非直接进行加减操作。

对于上述的情况，stack overflow上的一个[解释](https://stackoverflow.com/questions/50073792/whats-the-mechanism-behind-p2v-v2p-macro-in-xv6)，稍后再做分析。

