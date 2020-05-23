## 59 数据中心操作系统：Kubernetes

Kubernetes 在数据中心的作用就如同 Linux 在计算机的作用一样，被称为是数据中心操作系统。下面和 Linux 做一个对比：

![img](https://static001.geekbang.org/resource/image/49/47/497c8c2c0cb193e0380ed1d7c82ac147.jpeg)



Kubernetes主要还是管理四种硬件资源：

- CPU：依赖 Docker 和 Kubernetes 的 Scheduler 完成 CPU 的调度、亲和等。
- 内存：Docker 的 Namespace 和 CGroup
- 存储
- 网络



存储技术：

- 对象存储
- 分布式文件系统
- 分布式块存储



Kubernetes 网络模型：

- IP-Per-Pod：每个 Pod 都拥有一个独立 IP 地址，Pod 内所有容器共享一个网络命名空间
- 集群内所有 Pod 都在一个直接连通的扁平网络中，可通过 IP 直接访问。
- 所有容器之间无需 NAT 就可以直接互相访问
- 所有 Node 和 所有容器之间无需 NAT 就可以直接互相访问。
- 容器自己看到的 IP 跟其他容器看到的一样。



Kubernetes 的包管理软件是 Helm，类似 Linux 的 yum，apt 之类的。

Linux iptables -》Network Policy。

各自模块对比如下：

![img](https://static001.geekbang.org/resource/image/1a/e5/1a8450f1fcda83b75c9ba301ebf9fbe5.jpg)









