## 38 信号(下)

中断与信号的区别：

- 中断在内核驱动里注册中断处理函数
- 信号在用户态进程中注册信号处理函数

发送信号的系统调用：

- kill 或 sigqueue 发送信号给某个进程
- tkill 或 tgkill 发送信号给某个线程

最终都会调用到 `do_send_sig_info` 函数。



信号的发送和处理：

![img](https://static001.geekbang.org/resource/image/3d/fb/3dcb3366b11a3594b00805896b7731fb.png)

