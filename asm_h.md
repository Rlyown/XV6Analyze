[toc]

一些关于x86段描述符的宏定义。可以看到，它仅被*entry.S*，*bootasm.S*，*entryother.S*三个汇编文件引用。

段的数据结构在*mmu.h*中定义。



# SEG_NULLASM

清空一个段，从*mmu.h*中定义的段描述符长度为8个字节

```c
#define SEG_NULLASM                                             \
        .word 0, 0;                                             \
        .byte 0, 0, 0, 0
```

* `.word expression `：这个是汇编中的字，用于设置当前位置的值，[参考网站](https://blog.csdn.net/qq_40897531/article/details/106335620)；

  在这里的作用就是设置2个字（在x86中，1 word == 2 byte）为0。

* `.byte`：与word相同，它代表的单位是字节；

  这里是设置了4个字节为0



# SEG_ASM

通过*type, base, lim*设置一个段。

```c
#define SEG_ASM(type,base,lim)                                  \
        .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);      \
        .byte (((base) >> 16) & 0xff), (0x90 | (type)),         \
                (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```

* *(0x90 | (type))*修改的是` p, s, dpl, type`：9对应1001，即p为1，s为0，dpl为01
* *(0xC0 | (((lim) >> 28) & 0xf))*修改的是`g, db, rsv1, avl, limit(high 4 bit)`：C对应1100，即g为1，db为1，rsv1为0，avl为0
* 其余的很容易看出是设置base和limit