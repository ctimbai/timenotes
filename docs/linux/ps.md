## 35 块设备(下)

从文件系统到块设备的调用层次关系如下：



![img](https://static001.geekbang.org/resource/image/3c/0e/3c473d163b6e90985d7301f115ab660e.jpeg)

块设备IO 和字符设备IO一样，IO操作都分为两种：直接IO和缓存IO。

无论哪种IO，最终都会调用submit_bio 提交块设备的 IO 请求。

每一种块设备，都有一个 gendisk 表示这个设备，它有一个请求队列，这个请求队列是一系列的request对象，每个request对象里面包含多个BIO对象，指向page cache，所谓写入块设备，就是将 page cache 里的数据写入硬盘。

![img](https://static001.geekbang.org/resource/image/c9/3c/c9f6a08075ba4eae3314523fa258363c.png)

