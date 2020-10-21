[toc]

# pushcli/popcli

*pushcli*和*popcli*与*sti*和*cli*（定义在x86.h中）类似，但是它们的是严格对应的。执行了两次*pushcli*就必须执行两次*popcli*。

因此如果在执行*pushcli*之前，中断是关闭状态，那么执行*popcli*之后中断仍会是关闭的。

## pushcli

获取eflags寄存器信息，然后执行关中断函数cli()。

```c
void
pushcli(void)
{
  int eflags;
  
  eflags = readeflags();
  cli();
  if(cpu->ncli++ == 0)
    // 因为要求push和pop一一匹配，如果先执行了pop，则ncli < 0
    // 因此执行完push后，要还原中断状态
    cpu->intena = eflags & FL_IF;  // eflags & FL_IF的作用获取初始的中断状态
}
```



## popcli

与push的实现思路类似

```c
void
popcli(void)
{
  
  if(readeflags()&FL_IF)
    // 如果中断已经打开，则惩罚
    // 中断已开说明pop次数多于push
    panic("popcli - interruptible");
  if(--cpu->ncli < 0)
    // 如果pop次数多于push，则会导致ncli < 0
    panic("popcli");
  if(cpu->ncli == 0 && cpu->intena)
    // 仅在最后一个pop且初始状态是开中断的时候开中断
    // 这里就是保证push和pop匹配后，中断能回到初始状态的判据
    sti();
}
```



# initlock

初始化锁

```c
void
initlock(struct spinlock *lk, char *name)
{
  lk->name = name;
  lk->locked = 0;  // 表示该锁未被持有
  lk->cpu = 0;  // 没有CPU持有该锁
}
```



#  acquire

（日后解析）

```c
void
acquire(struct spinlock *lk)
{
  pushcli(); // disable interrupts to avoid deadlock.
  if(holding(lk))  // 有cpu
    panic("acquire");

  // The xchg is atomic.
  // It also serializes, so that reads after acquire are not
  // reordered before it. 
  // xchg返回的结果是eax寄存器的输出值
  // 这里就是在等待锁可用
  while(xchg(&lk->locked, 1) != 0)
    ;

  // Record info about lock acquisition for debugging.
  lk->cpu = cpu;
  getcallerpcs(&lk, lk->pcs);
}
```



# getcallerpcs

（日后解析）

```c
void
getcallerpcs(void *v, uint pcs[])
{
  uint *ebp;
  int i;
  
  ebp = (uint*)v - 2;
  for(i = 0; i < 10; i++){
    if(ebp == 0 || ebp < (uint*)KERNBASE || ebp == (uint*)0xffffffff)
      break;
    pcs[i] = ebp[1];     // saved %eip
    ebp = (uint*)ebp[0]; // saved %ebp
  }
  for(; i < 10; i++)
    pcs[i] = 0;
}
```



