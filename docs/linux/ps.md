## 29 虚拟文件系统

虚拟文件系统是为 Linux 访问底层系统提供一个统一的接口，Linux 可以支持多达数十种不同的文件系统。

Linux 一个文件操作的系统调用总的需要经历以下的几个层次：

![img](https://static001.geekbang.org/resource/image/3c/73/3c506edf93b15341da3db658e9970773.jpg)

挂载文件系统（mount）：

1. 挂载之前先注册：`register_filesystem`
2. mount 调用链：`do_mount->do_new_mount->vfs_kern_mount`
3. `mount_fs` 挂载文件系统，调用 `ext4_fs_type` 的 `ext4_mount` 函数挂载 `ext4` 文件系统

打开文件（open）：

1. open->do_sys_open -> `get_unused_fd_flags` 得到一个未使用的文件描述符
2. `path_init` 对文件路径进行查找，并逐层处理
3. `do_last` 实现目录项高速缓存，Linux为了提高目录项对象的处理效率，设计和实现了目录项高速缓存 dcache。
4. `vfs_open` 真正打开文件



对于每一个打开的文件，都有一个 dentry 对应，叫做 directory entry，不仅仅表示文件夹，也表示文件，指向文件对应的 inode。dentry 是放在一个 dentry cache 里，文件关闭了，它依然存在。inode 结构表示硬盘上的 inode，包括块设备号等。



## 30 文件缓存

脏页：写入到缓存，但是还没有写入到硬盘的页面。

缓存页：缓存以页为单位存在内存里，并以 `radix tree` 这种数据结构来存放。



read 或 write 系统调用：

1. 首先进入 VFS：`read->vfs_read->__vfs_read`
2. 每个文件有一个结构 `struct file` ，其中 `file_operations` 定义了文件系统层的操作，对于 ext4 文件系统来说，调用 `ext4_file_read_iter`
3. 这一步有两种方式，一种是直接读写硬盘，叫 直接 I/O，一种是通过缓存间接读写，叫 缓存 I/O
4. 如果是缓存 I/O ，会触发数据的预读和回写

具体过程用下面这张图来总结：

![img](https://static001.geekbang.org/resource/image/0c/65/0c49a870b9e6441381fec8d9bf3dee65.png)









