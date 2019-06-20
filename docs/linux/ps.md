## 35 块设备(下)

从文件系统到块设备的调用层次关系如下：



![img](https://static001.geekbang.org/resource/image/3c/0e/3c473d163b6e90985d7301f115ab660e.jpeg)

块设备IO 和字符设备IO一样，IO操作都分为两种：直接IO和缓存IO。

无论哪种IO，最终都会调用submit_bio 提交块设备的 IO 请求。

每一种块设备，都有一个 gendisk 表示这个设备，它有一个请求队列，这个请求队列是一系列的request对象，每个request对象里面包含多个BIO对象，指向page cache，所谓写入块设备，就是将 page cache 里的数据写入硬盘。

![img](https://static001.geekbang.org/resource/image/c9/3c/c9f6a08075ba4eae3314523fa258363c.png)



## 36 进程间通信

进程间通信的 几种方式：

- 管道：瀑布模型
- 消息队列：发邮件
- 共享内存+信号量：拉到会议室讨论
- 信号

### 管道

管道类似于软件开发中的瀑布模型，只有当上一下步骤完成才会开始下一个步骤，上个步骤的输出是下一个步骤的输入。

```sh
ps -ef | grep 关键字 | awk '{print $2}' | xargs kill -9
```



管道分为命名管道和匿名管道

创建命名管道：

```sh
mkfifo hello
```



查看管道：

```sh
# ls -l
prw-r--r--  1 root root         0 May 21 23:29 hello

# p 表示这个文件时管道文件
```



向管道一端输入东西：

```sh
# echo "hello world" > hello
```



从管道另一端查看东西：

```sh
# cat < hello 
hello world
```



### 消息队列

创建消息队列：

```sh
messagequeueid = msgget(key, IPC_CREAT | 0777)
```



每个消息队列都有一个唯一的 key，使用 ftok 生成。

```sh
key = ftok("/root/messagequeue/messagequeuekey", 1024)
```



发送消息：

```sh
msgsnd(messagequeueid, &buffer, len, IPC_NOWAIT)
# messagequeueid 指定发送消息的消息队列 ID
# buffer 消息结构体
# len 消息长度
# flag 消息标志 IPC_NOWAIT 表示发送消息时候不阻塞
```



接收消息：

```sh
msgrcv(messagequeueid, &buffer, 1024, type, IPC_NOWAIT)
# 1024 表示消息可接受的最大长度
# type 消息类型
```



使用命令行工具创建 IPC 对象：ipcmk，ipcs 和 ipcrm 分别用于创建、查看和删除 IPC 对象。



创建消息队列：

```sh
# ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x00016978 32768      root       777        0            0
```



### 共享内存

说的是多个进程共享同一块虚拟地址空间，映射到相同的物理空间。一个进程写入，另外的进程立马就能看到了。



创建共享内存：

```sh
int shmget(key_t key, size_t size, int flag);
# key 共享内存唯一的key
# size 共享内存的大小
# flag IPC_CREAT 创建一个新的共享内存
```



查看共享内存：

```sh
#ipcs --shmems
------ Shared Memory Segments ------ ­­­­­­­­
key        shmid    owner perms    bytes nattch status
0x00000000 19398656 marc  600    1048576 2      dest
```



将共享内存 attach 到虚拟地址空间的某个位置：

```sh
void *shmat(int shm_id, const void *addr, int flag);
# addr 表示 attach 的地址，通常设为 NULL，表示让内核来选一个合适的地址
```



删除共享内存(先解绑，再删除)：

```sh
# 解绑
int shmdt(void *addr); 

# 删除
int shmctl(int shm_id, int cmd, struct shmid_ds *buf);
```



### 信号量

保护共享内存中数据读写的冲突。

定义了两种原子操作：

- P 操作：申请资源，-N
- V 操作：归还资源，+N



创建信号量：

```sh
 int semget(key_t key, int num_sems, int sem_flags);
 # num_sems 表示创建多少个信号量，如果有多个资源需要管理，就可以创建一个信号量组
```



初始化信号量的总的资源数量：

```sh
int semctl(int semid, int semnum, int cmd, union semun args);
# semnum 信号量组中某个信号的 ID

union semun
{
  int val;
  struct semid_ds *buf;
  unsigned short int *array;
  struct seminfo *__buf;
};
```



无论P操作还是 V 操作，都使用 semop 函数：

```sh
int semop(int semid, struct sembuf semoparray[], size_t numops);


struct sembuf 
{
  short sem_num; // 信号量组中对应的序号，0～sem_nums-1
  short sem_op;  // 信号量值在一次操作中的改变量
  short sem_flg; // IPC_NOWAIT, SEM_UNDO
}
```



### 信号

异常情况下的进程间通信方式，比如线上系统故障等。

信号可以在任何时候发送给进程，进程需要为这个信号设置信号处理函数。相当于一个应急手册。



## 37 信号

Linux定义了非常多的信号，`kill -l` 可以查看所有的信号。

`man 7 signal` 查看每个信号的意思：

```sh
Signal     Value     Action   Comment
──────────────────────────────────────────────────────────────────────
SIGHUP        1       Term    Hangup detected on controlling terminal
                              or death of controlling process
SIGINT        2       Term    Interrupt from keyboard
SIGQUIT       3       Core    Quit from keyboard
SIGILL        4       Core    Illegal Instruction


SIGABRT       6       Core    Abort signal from abort(3)
SIGFPE        8       Core    Floating point exception
SIGKILL       9       Term    Kill signal
SIGSEGV      11       Core    Invalid memory reference
SIGPIPE      13       Term    Broken pipe: write to pipe with no
                              readers
SIGALRM      14       Term    Timer signal from alarm(2)
SIGTERM      15       Term    Termination signal
SIGUSR1   30,10,16    Term    User-defined signal 1
SIGUSR2   31,12,17    Term    User-defined signal 2
……
```



进程对信号的处理方式：

- 执行默认操作：比如 Core Dump，程序种植户，将进程状态保存到文件中，方式程序员事后分析问题
- 捕捉信号：执行信号处理函数
- 忽略信号



信号处理常见流程：

1. 注册信号处理函数
2. 发送信号



注册信号处理函数：

```sh
typedef void (*sighandler_t)(int);
int sigaction(int signum, const struct sigaction *act,
                     struct sigaction *oldact);
```



信号处理函数执行的动作保存在结构体 `struct sigaction` 中：

```sh
struct sigaction {
	__sighandler_t sa_handler;
	unsigned long sa_flags;
	__sigrestore_t sa_restorer;
	sigset_t sa_mask;		/* mask last for extensibility */
};
```



sigaction 的调用过程：

```sh
sigaction-> __sigaction -> __libc_sigaction -> rt_sigaction(内核) -> do_sigaction
```



大体的流程如下所示：

![img](https://static001.geekbang.org/resource/image/7c/28/7cb86c73b9e73893e6b0e0433d476928.png)

