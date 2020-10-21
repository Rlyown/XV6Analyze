[toc]



# ptable

process table，进程表，由一个自旋锁和一个进程数组组成（*NPROC*定义在*param.h*文件中，值为64）

```c
struct {
  struct spinlock lock;
  struct proc proc[NPROC];
} ptable;
```



##pinit

初始化ptable的锁。

```c
void
pinit(void)
{
  initlock(&ptable.lock, "ptable");
}
```

*initlock*函数实现在*spinlock.c*中，用于初始化一个自旋锁的参数。



# allocproc

在ptable中寻找一个未被使用的进程（UNUSED），如果找到则将其状态修改为初始（EMBRYO），并且初始化内核中的一些状态；反之则返回0。

*allocproc*主要工作如下：

1. 在进程表中寻找未使用的进程，
2. 为该进程申请一页的空间用作栈
3. 在栈中为*trapframe*、*trapret*、*context*设置初始位置
4. 初始化*context*部分内存的数值
5. 将*context*中的*eip*寄存器存储*forkret*

```c
static struct proc*
allocproc(void)
{
  struct proc *p;
  char *sp;

  acquire(&ptable.lock);  // 请求锁
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++)
    if(p->state == UNUSED)
      goto found;  // 找到就跳转
  release(&ptable.lock);
  return 0;

found:
  p->state = EMBRYO;  // 修改状态为初始状态
  p->pid = nextpid++;  // 分配pid，nextpid从1开始自增
  release(&ptable.lock);

  // Allocate kernel stack.
  // 调用kalloc从空闲页表中获取一页
  if((p->kstack = kalloc()) == 0){
    // 获取失败，放弃进程的创建
    p->state = UNUSED;
    return 0;
  }
  sp = p->kstack + KSTACKSIZE;  // 进程的内核栈底指针 + 内核栈大小
  
  // Leave room for trap frame.
  sp -= sizeof *p->tf;  // *p->tf是当前进程的trap帧
  p->tf = (struct trapframe*)sp;  // 此时p->tf指向trapframe所在的栈地址
  
  // Set up new context to start executing at forkret,
  // which returns to trapret.
  sp -= 4;
  *(uint*)sp = (uint)trapret;

  sp -= sizeof *p->context;
  p->context = (struct context*)sp;
  memset(p->context, 0, sizeof *p->context);
  p->context->eip = (uint)forkret;

  return p;
}
```



# userinit

设置第一个用户进程。

```c
void
userinit(void)
{
  struct proc *p;
  extern char _binary_initcode_start[], _binary_initcode_size[];
  
  p = allocproc();
  initproc = p;
  if((p->pgdir = setupkvm()) == 0)  // 构建内核映射
    panic("userinit: out of memory?");
  inituvm(p->pgdir, _binary_initcode_start, (int)_binary_initcode_size);  // 装载initcode
  p->sz = PGSIZE;
  memset(p->tf, 0, sizeof(*p->tf));
  p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
  p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
  p->tf->es = p->tf->ds;
  p->tf->ss = p->tf->ds;
  p->tf->eflags = FL_IF;
  p->tf->esp = PGSIZE;
  p->tf->eip = 0;  // beginning of initcode.S

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;
}
```

这里引用了两个符号*_binary_initcode_start*和*_binary_initcode_size*，这两个符号是来自汇编定义的。

* `_binary`：表明它来源于一个汇编代码

* `_initcode`：文件名字

* `_start, _end, _size`：特殊的符号。如下是[man objcopy](https://linux.die.net/man/1/objcopy)中的解释，以及一个stack overflow上的讨论[binutils - kernel - “_binary” meaning?](https://stackoverflow.com/questions/29034840/binutils-kernel-binary-meaning)

  > You can access this binary data inside a program by referencing the special symbols that are created by the conversion process. These symbols are called _binary_*objfile*_start, _binary_*objfile*_end and _binary_*objfile*_size. e.g. you can transform a picture file into an object file and then access it in your code using these symbols.



#growproc

修改进程内存大小。n > 0表示增加内存n字节；n < 0则表示减少内存n字节

```c
int
growproc(int n)
{
  uint sz;
  
  sz = proc->sz;
  if(n > 0){
    if((sz = allocuvm(proc->pgdir, sz, sz + n)) == 0)
      return -1;
  } else if(n < 0){
    if((sz = deallocuvm(proc->pgdir, sz, sz + n)) == 0)
      return -1;
  }
  proc->sz = sz;
  switchuvm(proc);
  return 0;
}
```



# fork

创建一个新的进程，将p复制为父进程。设置堆栈以使其好像从系统调用中返回一样。调用者必须将返回的proc的状态设置为RUNNABLE。

```c
int
fork(void)
{
  int i, pid;
  struct proc *np;

  // Allocate process.
  if((np = allocproc()) == 0)
    return -1;

  // Copy process state from p.
  if((np->pgdir = copyuvm(proc->pgdir, proc->sz)) == 0){
    kfree(np->kstack);
    np->kstack = 0;
    np->state = UNUSED;
    return -1;
  }
  np->sz = proc->sz;
  np->parent = proc;
  *np->tf = *proc->tf;

  // Clear %eax so that fork returns 0 in the child.
  np->tf->eax = 0;

  for(i = 0; i < NOFILE; i++)
    if(proc->ofile[i])
      np->ofile[i] = filedup(proc->ofile[i]);
  np->cwd = idup(proc->cwd);
 
  pid = np->pid;
  np->state = RUNNABLE;
  safestrcpy(np->name, proc->name, sizeof(proc->name));
  return pid;
}
```