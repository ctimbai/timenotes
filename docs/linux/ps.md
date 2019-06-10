## 32 字符设备(上)

打开一个字符设备，需要经历以下的一些过程：

1. 把写好的内核模块 （.ko 结尾），通过 insmod 加载进内核，`module_init`
2. 通过 mknod 在 /dev 下创建一个设备文件，并和相应的设备号进行关联，最后就可以通过文件系统的接口，对这个设备文件进行操作。
3. open 打开设备文件



![img](https://static001.geekbang.org/resource/image/2e/e6/2e29767e84b299324ea7fc524a3dcee6.jpeg)

打开设备文件之后，就可以对文件进行读写了。

通过 read write 调用到内核中的 `sys_read` 和 `sys_write` 进行操作。

除此之外，还可以调用 `ioctl` 进行一些特殊的 I/O 操作。



