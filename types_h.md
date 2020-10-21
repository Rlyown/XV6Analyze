该文件定义了一些数据类型的简写，或是别名。

这其中比较重要的就是*pde_t*类型，它是页目录/页表项（32 bit）的类型

```c
typedef unsigned int   uint;
typedef unsigned short ushort;
typedef unsigned char  uchar;
typedef uint pde_t;

```

