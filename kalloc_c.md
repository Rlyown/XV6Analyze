[toc]



# kinit1

初始化自旋锁，但暂时关闭了锁的使用。并执行*freerange()*。此时在*freerange()*中会调用*kfree*，这时候的调用并不是释放空间，而是用于初始化分配器。

```c
void
kinit1(void *vstart, void *vend)
{
  initlock(&kmem.lock, "kmem");
  kmem.use_lock = 0;
  freerange(vstart, vend);
}
```



# kinit2



```c
void
kinit2(void *vstart, void *vend)
{
  freerange(vstart, vend);
  kmem.use_lock = 1;
}
```



# freerange

该函数先对起始地址进行对其操作（*PGROUNDUP*），此时p指向了第一个页的地址，然后在循环中按页遍历，逐个释放页的空间。

```c
void
freerange(void *vstart, void *vend)
{
  char *p;
  p = (char*)PGROUNDUP((uint)vstart);
  for(; p + PGSIZE <= (char*)vend; p += PGSIZE)
    kfree(p);
}
```



# kfree

大致意思是将传入的页（变量v）内的空间全部置1，然后再放入*kmem*的空闲列表的首部。

这个变量v通常是使用*kalloc*函数返回页，但是也有例外，即在*kinit*中用于初始化分配器。

```c
void
kfree(char *v)
{
  struct run *r;

  if((uint)v % PGSIZE || v < end || V2P(v) >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(v, 1, PGSIZE);

  if(kmem.use_lock)
    acquire(&kmem.lock);
  r = (struct run*)v;
  r->next = kmem.freelist;
  kmem.freelist = r;
  if(kmem.use_lock)
    release(&kmem.lock);
}
```



# kalloc

与kfree相对应，用于分配空间，取出一页的地址并返回。如果分配失败，则返回0。

```c
char*
kalloc(void)
{
  struct run *r;

  if(kmem.use_lock)
    acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  if(kmem.use_lock)
    release(&kmem.lock);
  return (char*)r;
}
```

