[toc]



## 01 如何学习 Linux 性能优化

首先了解 **性能指标** 的概念：

高并发+响应快 = 吞吐+延时

一般从应用负载和系统资源的视角来分别看待：

![](images/perfindex.png)



从分层的角度来看，每一层都对应不同的指标要求。

其次是学习的重点：懂得基本基础，把握整体系统性能的全局观，不要深陷细节。

最后善于总结、分类。

![](images/alltools.png)



# CPU

CPU 性能指标常用有：

- 平均负载
- CPU 上下文切换
- CPU 使用率



## 02 理解平均负载



**平均负载**：指单位时间内，系统处于 **可运行状态（R）** 和 **不可中断状态（D）** 的平均进程数，也就是 **平均活跃进程数** 。

- **可运行状态：** 正在使用 CPU 或者正在等待 CPU 的进程
- **不可中断状态：** 等待 IO 响应、磁盘读写等



查看命令：`uptime` 或者 `top`

```sh
# uptime
02:34:03 up 2 days, 20:14, 1 user, load average: 0.63, 0.83, 0.88
```

最后三个数字，分别表示 **过去**  1 分钟、5 分钟、15 分钟的平均负载。



查看CPU的个数：  
```shell
grep "model name" /proc/cpuinfo | wc -l
```



**经验之谈：** 当平均负载高于 CPU 数量 **70%** 的时候，就应该分析负载高的问题了。还有就是判断负载的趋势变化。



### 平均负责与 CPU 使用率

- 平均负载：不仅包括了正在使用 CPU 的进程，还包括等待 CPU 和 等待 IO 的进程
- CPU 使用率：单位时间内 CPU 繁忙情况的统计，跟平均负责并不一定完全对应，有以下一些联系：
  - CPU 密集型进程：使用大量 CPU 会导致平均负载升高，此时两者等价
  - IO 密集型进程：等待 IO 也会导致平均负载升高，但 CPU 使用率并不一定会升高
  - 大量等待 CPU 的进程调度也会导致平均负载升高，CPU 使用率也会比较高



### 实验



**工具：**

- 工具包：`stress` 和 `sysstat`
- `stress` 是一个Linux系统压力测试工具
- `sysstat` 包含了常用的 Linux 性能分析工具，如：`mpstat` 和 `pidstat`
- `mpstat` 多核CPU性能分析功能，用于实时查看每个CPU的性能指标，以及所有CPU的平均指标
- `pidstat` 进程性能分析工具，用来实时查看进程的CPU、内存、I/O以及上下文切换等性能指标



**常用命令：**

```sh
# 模拟一个 CPU 使用率 100% 的场景
stress --cpu 1 --timeout 600

# 模拟 I/O 压力，不断执行 sync
stress -i 1 --timeout 600

# 模拟超配的场景，进程数超过 CPU 个数
stress -c 8 --timeout 600

# mpstat 查看 CPU 使用率的变化情况
# -P ALL 监控所有 CPU，后面数字 5 标识每间隔 5s 输出一组数据
mpstat -P ALL 5

# pidstat 查看哪个进程导致了 CPU 使用率 100% 
pidstat -u 5 1
```



> 注意：
>
> 1 无法看到 iowait 升高
>
> 解决办法：使用 stress-ng：stress-ng -i 1 --hdd 1 --timeout 600（--hdd 表示读写临时文件）
>
> 2 pidstat 输出没有 %wait
>
> 解决办法：需要升级 sysstat 包 到 11.5.5 版本以后
>
> 3 mpstat 无法观测
>
> 解决办法：等待时间长一些，比如 20s：mpstat -P ALL 5 20



## 03 CPU 上下文切换

CPU 上下文：包括**CPU 寄存器** 和 **程序计数器**。前者是 CPU 内置的容量小、但速度极快的内存；后者是用来存储 CPU 正在执行的指令位置、或者即将执行的吓一条指令位置。

包括：进程上下文切换、线程上下文切换以及中断上下文切换。

CPU 上下文切换本质是 CPU 特权级的切换。

**进程上下文**切换发生在内核态，不仅包括虚拟内存、栈、全局变量等用户空间的资源，还包括了内核堆栈、寄存器等内核空间的状态。

**线程的上下文**切换包含两种情况：一两个线程属于不同的进程，这种情况和进程的上下文切换一样，二两个线程属于同一个进程，由于虚拟内存、全局变量是共享的，所以只切换线程的私有数据（栈）、寄存器等不共享的数据。

### 查看 CPU 上下文切换

```
# 每隔 5 秒输出 1 组数据
$ vmstat 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 3  0      0 161484 134588 1522148    0    0     0     2   22   20  0  0 100  0  0
```

- `cs(context switch)` 每秒上下文切换的次数
- `in(interrupt)` 每秒中断次数
- `r(running or runnable)` 就绪队列长度，也就是正在运行和等待 CPU 的进程数
- `b(Blocked)` 处于不可中断睡眠状态的进程数 

### 查看进程上下文切换次数

```
# 每隔 5 秒输出 1 组数据 
$ pidstat -w 5
Linux 3.10.0-862.14.4.el7.x86_64 (by) 	12/18/2018 	_x86_64_	(1 CPU)

08:58:15 AM   UID       PID   cswch/s nvcswch/s  Command
08:58:15 AM     0         1      0.05      0.00  systemd
08:58:15 AM     0         2      0.00      0.00  kthreadd
...
```

- `cswch(voluntary context swithches)` 每秒自愿上下文切换（进程无法获取所需资源而发生）
- `nvcswch(non voluntary context switches)` 每秒非自愿上下文切换（进程由于时间片已到等原因，被系统强制调度）


以上默认显示的是进程的指标数据，如果要查看线程数据，则需要加上 -t 参数。

```
# -wt 表示输出线程上下文切换指标
$ pidstat -wt 1
Linux 3.10.0-862.14.4.el7.x86_64 (by) 	12/21/2018 	_x86_64_	(1 CPU)

08:57:17 AM   UID      TGID       TID   cswch/s nvcswch/s  Command
08:57:18 AM     0         3         -      2.02      0.00  ksoftirqd/0
08:57:18 AM     0         -         3      2.02      0.00  |__ksoftirqd/0
```

### 模拟系统上下文切换工具

**sysbench** : 一个多线程基准测试工具，可以用来模拟系统多线程调度切换

ubuntu 安装
```
apt install sysbench sysstat
```

模拟系统多线程调度
```
# 以 10 个线程运行 5 分钟基准测试，模拟多线程切换问题
sysbench --threads=10 --max-time=300 threads run 
```

### 查看引起频繁中断的中断类型
查看 `/proc/interrupts`

使用 `watch -d cat /proc/interrupts` 查看中断的变化情况，重点关注 `RES` 行，重调度中断，表示唤醒空闲状态的 CPU 来调度新的任务运行，这是 SMP 系统中，调度器用来分散任务到不同 CPU 的机制。

**结论：**

每秒切换次数在数百-一万且切换次数稳定都算是正常，如果超过一万次，或者切换次数出现数量级增长时就不稳定。

## 04 CPU 使用率
计算公式：  
```
CPU 使用率=1-空闲时间/总CPU时间
```

为了计算准确，一般会算时间差：  
```
平均CPU使用率=1-(空闲时间new - 空闲时间old)/(总CPU时间new - 总CPU时间old)
```

### 查看 CPU 使用率
- `top` 显示系统总体的 CPU 和内存使用情况，以及各个进程的资源使用情况（默认没3s刷新一次，按下 1 查看单个 CPU 的使用率）
- `ps` 只显示每个进程的资源使用情况（整个进程生命周期）
- `pidstat` 查看每个进程的详细 CPU 使用率
- `perf top` 实时显示占用CPU时钟最多的函数或指令
- `perf record` 比 `perf top` 多了保存采样数据的环节
- `perf report` 显示 `perf record` 的报告



## 05 不可中断进程和僵尸进程

进程的状态：

- R：进程在 CPU 就绪队列中，正在运行或等待运行
- D（Disk Sleep）：不可中断状态睡眠
- Z（Zombie）：僵尸进程
- S（Inerruptible Sleep）：可中断状态睡眠
- I（Idle）：空闲状态
- T（Traced）：表示进程处于暂停或跟踪状态
- X（Dead）：表示进程已消亡



iowait 分析

推荐使用 dstat，它吸收了 vmstat、iostat、ifstat 等几种工具的优点。可以同时查看 CPU、I/O、网络和内存的使用情况，便于对比分析。

查看某一个进程的磁盘读写情况：

```sh
pidstat -d -p 4344 1 3
# -d 表示统计 I/O 数据，-p 指定进程号
```

strace 跟踪系统进程系统调用的工具。先通过 `pidstat -d` 找出影响进程 id，再用 strace 来跟踪。



## 缓存.
### 缓存命中情况
工具：

- cachestat 提供了整个操作系统缓存的读写命中情况
- cachetop 提供每个进程的缓存命中情况

### 网友总结的性能排查思路
1. 有监控的时候，看监控大盘
2. 没有的情况下，首先看系统的平均负载（`top` 或 `htop`），标示这系统的一个整体情况
3. 平均负载高，看具体每个CPU核的使用情况（`top 按1`），如果占比很高，那瓶颈应该是CPU，接下来看是什么进程导致的（`pidstat`）
4. 如果CPU没问题，就去看内存，首先 `free` 内存的使用情况，结合看 cache 和 buffer，再看什么进程占用了过高的内存（`top 排序`）
5. 内存没问题就看磁盘（`iostat`）
6. 还有带宽问题，使用 `iftop` 查看流量使用情况，看流量是否超过本机给定带宽
7. 涉及到具体应用，根据应用的参数来查看，比如连接数是否超过设定值等
8. 如果系统层各个指标查下来都没有发现异常，那么久要考虑外部系统，比如数据库、缓存、存储等。

# 网络

从下往上：

|    测量指标     |          测量工具           |  测量单位  |
| :-------------: | :-------------------------: | :--------: |
|  应用负载性能   |             wrk             |    #/s     |
|  HTTP性能 QPS   |         ab/webbench         |    #/s     |
|  TCP/UDP吞吐量  |       iperf3/netperf        |    Mb/s    |
|    网络层PPS    |        pktgen/hping3        |    包/s    |
| 套接字状态/队列 |        netstat -lntp        |     无     |
| 协议栈统计信息  |      netstat -s/ss -s       |     无     |
|  网卡统计信息   | ifconfig/ip -s addr show xx |     无     |
|   网络吞吐量    |        sar -n DEV 1         |    KB/s    |
|       PPS       |        sar -n DEV 1         |    包/s    |
|    网络带宽     |  ethtool eth0\|grep Speed   | Gb/s, Mb/s |
|    链路时延     |            ping             |     s      |



## 33 Linux 网络基本模型

### 网络模型（OSI | TCP/IP）

- OSI 开放互联七层模型
- TCP/IP 四层模型



![img](https://static001.geekbang.org/resource/image/f2/bd/f2dbfb5500c2aa7c47de6216ee7098bd.png)





网络常见技术：

- 七层负载均衡
- 四层负载均衡
- 三层设备
- 二层设备





### Linux 网络栈

Linux 通用 IP 网络栈：

![img](https://static001.geekbang.org/resource/image/c7/ac/c7b5b16539f90caabb537362ee7c27ac.png)



### Linux 网络收发流程

总体图示：

![img](https://static001.geekbang.org/resource/image/3a/65/3af644b6d463869ece19786a4634f765.png)



左边是收包流程，右边是发包流程。

**收包流程：** 

1. 网卡收到一个网络包后，会通过 DMA 的方式，将这个包送到收包队列中
2. 然后通过 **硬中断** ，告诉中断处理程序已经收到包
3. 中断处理程序会为这个包分配内核数据结构（sk_buff），并将其拷贝到 sk_buff 缓冲区中
4. 然后再通过 **软中断** ，通知内核协议栈收到了新的包
5. 内核协议栈从缓冲区中取出这个包，通过网络协议栈，逐层处理这个包
   - **链路层** 检查报文的合法性，找出上层协议（IPv4 还是 IPv6），解封装，去掉帧头和帧尾，交给网络层。
   - **网络层** 取出 IP 头，判断这个包的下一步走向，是转发还是继续交给上层处理，如果确认是发给本机的包，那么就检查上层协议的类型（是 UDP 还是 TCP），同样解封装，去掉 IP 头，交给传输层。
   - **传输层** 取出 TCP 头或 UDP 头，根据  **<源IP、源端口、目的IP、目的端口>** 四元组，找出对应的 socket，把数据拷贝到 socket 的接收缓冲区中。
   - **应用层** 程序就可以使用 socket  接口，读取到接收到的数据了。



**发包流程：**

1. 应用程序调用 socket API 发送网络包
2. 由于系统调用会陷入到内核的套接字层中，套接字层会将数据放到 socket 发送缓冲区中
3. 网络协议栈从 socket 缓冲区取出数据，再按照 协议栈，自上而下地逐层处理数据包。
4. 这一切完成后，数据包被送到发送队列中，会触发软中断通知驱动程序，有包需要发送。
5. 驱动程序再通过 DMA ，从发包队列中读出网络包，继而通过物理网卡发送出去。



## 34 Linux 网络性能分析指标和工具

### 性能指标

说到网络性能优化，有哪些可衡量的指标呢？

常用的指标无外乎四个：带宽、吞吐量、延时、PPS（Packet Per Second）。

- **带宽：** 表示链路最大的传输速率，单位通常是 b/s（比特/s）
- **吞吐量：** 表示单位时间内成功传输的数据量，单位通常是 b/s（比特/s） 或 B/s（字节/秒），
- **延时：** 表示从网络请求发出后，一直到收到远端响应，所需要的时间。
- **PPS：** 表示以网络包为单位的传输速率，单位是 Packet/s（包/秒），用以衡量网络的转发能力。



除了以上四个常用的指标之外，还有：

- 网络的可用性（网络是否能正常通信）
- 并发连接数（TCP连接数量）
- 丢包数（丢包百分比）
- 重传率（重新传输的网络包比例）



### 测量工具

记住：从下到上

链路-》网卡-》协议栈-》应用程序

**网络接口** 统计信息使用工具：

- ip
- ifconfig

**网络协议栈** 统计信息使用工具

- netstat
- ss：比 netstat 速度更快

**网络吞吐和 PPS** 使用工具：

- sar -n

**网络带宽** 使用工具：

- ethtool (ethtool eth0 | grep speed)

**网络连通性和时延** 使用工具：

- ping



网络连接情况分析例子：

```sh
# head -n 3 表示只显示前面 3 行
# -l 表示只显示监听套接字
# -n 表示显示数字地址和端口 (而不是名字)
# -p 表示显示进程信息
$ netstat -nlp | head -n 3
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      840/systemd-resolve

# -l 表示只显示监听套接字
# -t 表示只显示 TCP 套接字
# -n 表示显示数字地址和端口 (而不是名字)
# -p 表示显示进程信息
$ ss -ltnp | head -n 3
State    Recv-Q    Send-Q        Local Address:Port        Peer Address:Port
LISTEN   0         128           127.0.0.53%lo:53               0.0.0.0:*        users:(("systemd-resolve",pid=840,fd=13))
LISTEN   0         128                 0.0.0.0:22               0.0.0.0:*        users:(("sshd",pid=1459,fd=3))
```

当套接字处于连接状态（Established）时:

- Recv-Q 表示套接字缓冲还没有被应用程序取走的字节数（即接收队列长度）
- Send-Q 表示还没有被远端主机确认的字节数（即发送队列长度）

当套接字处于监听状态时：

- Recv-Q 表示 syn backlog 的当前值
- Send-Q 表示最大的 syn backlog 值

syn backlog 是 TCP 协议栈中的半连接队列长度，相应的也有一个全连接队列（accept queue），它们都是维护 TCP 状态的重要机制。



协议栈统计信息分析例子：

```sh
$ netstat -s
...
Tcp:
    3244906 active connection openings
    23143 passive connection openings
    115732 failed connection attempts
    2964 connection resets received
    1 connections established
    13025010 segments received
    17606946 segments sent out
    44438 segments retransmitted
    42 bad segments received
    5315 resets sent
    InCsumErrors: 42
...

$ ss -s
Total: 186 (kernel 1446)
TCP:   4 (estab 1, closed 0, orphaned 0, synrecv 0, timewait 0/0), ports 0

Transport Total     IP        IPv6
*	  1446      -         -
RAW	  2         1         1
UDP	  2         2         0
TCP	  4         3         1
...
```

- ss 只显示已经连接、关闭、孤儿套接字等简要统计
- netstat 则提供的是更详细的网络协议栈信息



网络吞吐和 PPS 分析例子：

```sh
# 数字 1 表示每隔 1 秒输出一组数据
$ sar -n DEV 1
Linux 4.15.0-1035-azure (ubuntu) 	01/06/19 	_x86_64_	(2 CPU)

13:21:40        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
13:21:41         eth0     18.00     20.00      5.79      4.25      0.00      0.00      0.00      0.00
13:21:41      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
13:21:41           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

- rxpck/s 和 txpck/s 分别是接收和发送的 **PPS**。
- rxkB/s 和 txkB/s 分别是接收和发送的 **吞吐量**，单位是 KB/秒
- rxcmp/s 和 txcmp/s 分别是接收和发送的 **压缩数据包数**，单位是包/秒。
- %ifutil 是网络接口的使用率，即半双工模式下为 (rxkB/s+txkB/s)/Bandwidth，而全双工模式下为 max(rxkB/s, txkB/s)/Bandwidth



网络带宽（Bandwidth）分析例子：

用 `ethtool` 工具查询，单位通常是 Gb/s 或者 Mb/s（小写 b），查询网卡带宽：

```sh
$ ethtool eth0 | grep Speed
	Speed: 1000Mb/s
```

连通性和延时分析例子：

通常使用 ping：

```sh
# -c3 表示发送三次 ICMP 包后停止
$ ping -c3 114.114.114.114
PING 114.114.114.114 (114.114.114.114) 56(84) bytes of data.
64 bytes from 114.114.114.114: icmp_seq=1 ttl=54 time=244 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=47 time=244 ms
64 bytes from 114.114.114.114: icmp_seq=3 ttl=67 time=244 ms

--- 114.114.114.114 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 244.023/244.070/244.105/0.034 ms
```

## 35 C10K 、C100K、C10M 问题

- C10K：Linux 2.6中引入的 epoll 机制，可以完美解决
- C100K：在前基础上，增加系统的物理资源，可以满足
- C1000K：在前基础上，要对系统的软硬件进行统一优化，包括从硬件的中断处理，到网络协议栈的文件描述符数量、连接跟踪状态、缓存队列、再到应用程序的工作模型等的整个网络链路，都需要深入优化
- C10M：使用 bypass 旁路内核的方案，比如 XDP 或者 DPDK

实际上，在大多数场景中，我们并不需要单机并发 1000 万请求。通过调整系统架构，把请求分发到多台服务器中并行处理，才是更简单、扩展性更好的方案。

## 36 套路：怎么评估系统的网络性能？

针对不同的应用程序问题，采用自下而上的方法来分析性能瓶颈来源。

网络接口层和网络层的转发性能：

- 衡量性能指标是 PPS
- 使用工具：hping3、pktgen

TCP/UDP 性能（传输层）：

- 衡量性能指标是 吞吐量
- 使用工具：iperf3、netperf

HTTP 性能（应用层）：

- 测试方法：压力测试
- 衡量性能指标是 服务的每秒请求数（QPS）、请求延迟、吞吐量以及请求延迟的分布情况等
- 使用工具：ab、webbench

应用负载性能（应用程序的实际性能）：

- 衡量性能指标是 应用程序的请求负载
- 使用工具：wrk、TCPCopy、Jmeter、LoadRunner 等

## 37 DNS 解析性能优化

两个常用的工具：

- nslookup：如果解析时间较长，要看 DNS 服务器的地址是否距离太远导致丢包，比如 Google 的 8.8.8.8 就比电信提供的 114.114.114.114 要远
  - nslookup -debug 调试
  - time nslookup 查询解析时间
- dig
  -  dig +trace +nodnssec time.geekbang.org @8.8.8.8 （@指定DNS服务器）

有时候多次执行 nslookup，会发现有时候非常耗时。

增加缓存可以减少 DNS 解析的效率，我们使用的主流 Linux 发行版，除了最新版本的 Ubuntu （如 18.04 或者更新版本）外，其他版本并没有自动配置 DNS 缓存。想要为系统开启 DNS 缓存，就需要你做额外的配置。比如，最简单的方法，就是使用 dnsmasq。

 启动 dnsmasq： 

```sh
/# /etc/init.d/dnsmasq start
 * Starting DNS forwarder and DHCP server dnsmasq  [ OK ]
```

然后，修改 /etc/resolv.conf，将 DNS 服务器改为 dnsmasq 的监听地址，这儿是 127.0.0.1。接着，重新执行多次 nslookup 命令：

```sh
/# echo nameserver 127.0.0.1 > /etc/resolv.conf
/# time nslookup time.geekbang.org
Server:    127.0.0.1
Address:  127.0.0.1#53

Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.176

real  0m0.492s
user  0m0.007s
sys  0m0.006s

/# time nslookup time.geekbang.org
Server:    127.0.0.1
Address:  127.0.0.1#53

Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.176

real  0m0.011s
user  0m0.008s
sys  0m0.003s
```

现在我们可以看到，只有第一次的解析很慢，需要 0.5s，以后的每次解析都很快，只需要 11ms。并且，后面每次 DNS 解析需要的时间也都很稳定。

```sh
# centos 7 dnsmasq如下操作：

cat /etc/resolv.conf
nameserver 114.114.114.114

yum -y install dnsmasq
systemctl start dnsmasq

测试dns缓存，要测试查询速度，请访问一个 dnsmasq 启动后没有访问过的网站，执行：
[root@node ~]# dig archlinux.org | grep "Query time"
;; Query time: 212 msec
[root@node ~]# dig archlinux.org | grep "Query time"
;; Query time: 2 msec
再次运行命令，因为使用了缓存，查询时间应该大大缩短。
```

## 38 tcpdump 和 wireshark 分析网络流量

如果 ping 每次来回的时间都较短，但总的时间却花销较大，大概率是解析 PTR 的原因，PTR 表示进行域名的反向解析，这种情况需要加 ping 选项 -n 禁止域名解析。

tcpdump 的常用选项和过滤项总结成以下两张表：

![](images/tcpdump1.png)

![](images/tcpdump2.png)

wireshark 展示交互图的方法:

从菜单栏中，点击 Statistics -> Flow Graph，然后，在弹出的界面中的 Flow type 选择 TCP Flows，你可以更清晰的看到，整个过程中 TCP 流的执行过程：

![](images/wiresharkflow.png)

其中，断开连接的过程只有三次挥手，这是因为服务器端收到客户端的 FIN 后，服务器端同时也要关闭连接，这样就可以把 ACK 和 FIN 合并到一起发送，节省了一个包，变成了“三次挥手”。

**根据 IP 地址反查域名、根据端口号反查协议名称，是很多网络工具默认的行为，而这往往会导致性能工具的工作缓慢。** 所以，通常，网络性能工具都会提供一个选项（比如 -n 或者 -nn），来禁止名称解析。

## 39 怎么缓解 DDoS 攻击带来的性能下降问题

只是缓解，而不能解决 DDoS 问题。



## 40 如何优化网络请求延迟

延迟可能来源于三个方面：

- 网络延迟
- 协议栈延迟
- 应用程序处理延迟

分析网络延迟的工具：ping、hping3、traceroute

因为网络上的防火墙一般会过滤 ICMP 包，所以一般使用 `hping3 -S` 或 `traceroute --tcp` 使用 TCP SYN 包进行测量。

对于协议栈的优化，注意 TCP 的延迟确认机制和 `tcp_nodelay` 机制有时很影响性能。

## 41/42 优化 NAT 性能

iptables 中的 conntrack，表示连接跟踪模块。它通过内核中的连接跟踪表（也就是哈希表），记录网络连接的状态，是 iptables 状态过滤（-m state）和 NAT 的实现基础。

iptables 中关于 NAT 的三个动作：SNAT、MASQUERADE、DNAT。

其中 SNAT 和 DNAT 好理解，SNAT 和 MASQUERADE 区别是：

- SNAT 配置具体的 IP 地址
- MASQUERADE 配置一个子网

比如：

SANT：

```sh
$ iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100
```

MASQUERADE:

```sh
$ iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j MASQUERADE
```



## 43/44 套路：网络优化的思路



## 55 分析性能的一般套路

### 系统资源瓶颈

### 应用程序瓶颈

# 综合性能分析

常见的丢包场景：（从协议栈自下而上进行分析）

1. 网络传输：在两台主机连接之间，可能出现网络拥塞、线路错误等导致的传输失败而丢包
2. 物理层：网卡收包后，环形缓冲区可能会因为溢出而丢包
3. 链路层：可能会因为网络帧校验失败、QoS （配置 TC 策略）等而丢包
4. 网络层：可能会因为路由失败、组包大小超过MTU等而丢包
5. 传输层：可能会因为端口未监听、资源占用超过内核限制等而丢包
6. 套接字层：可能会因为套接字缓冲区溢出而丢包
7. 应用层：可能因为应用程序异常而丢包
8. 另外，如果配置了 iptables 规则，也会因为 iptables 规则而导致丢包

