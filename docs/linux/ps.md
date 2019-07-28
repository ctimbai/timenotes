## 50 计算虚拟化之 CPU

本节来看下用户态的 Qemu 和 内核态的 kvm 如何一起协作，来创建虚拟机，实现 CPU 的虚拟化。

Qemu 启动的一般命令：

```sh
qemu-system-x86_64 -enable-kvm -name ubuntutest  -m 2048 -hda ubuntutest.qcow2 -vnc :19 -net nic,model=virtio -nettap,ifname=tap0,script=no,downscript=no
```



