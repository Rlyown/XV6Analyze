这里则是定义了一些全局的常量，与之类似的是Linux中的ulimit命令所见到的量。

这些常量对系统的资源利用作出了限制。

```c
#define NPROC        64  // 最大进程数量
#define KSTACKSIZE 4096  // 每个处理内核
#define NCPU          8  // 最大CPU数量
#define NOFILE       16  // 每个进程可以开启的文件数量
#define NFILE       100  // 每个系统可以开启的文件数量
#define NBUF         10  // 磁盘块缓冲区大小
#define NINODE       50  // 最大的活跃i-node数量
#define NDEV         10  // 最大的主设备号
#define ROOTDEV       1  // 文件系统根磁盘的设备号
#define MAXARG       32  // 最大执行参数
#define LOGSIZE      10  // 磁盘日志中的最大数据扇区

```



