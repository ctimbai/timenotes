## page cache 问题

page cache 属于内核管理的内存，在内核中处于下列的位置：

![](./images/pagecache.png)

查看 page cache：

```sh

$ cat /proc/meminfo
...
Buffers:            1224 kB
Cached:           111472 kB
SwapCached:        36364 kB
Active:          6224232 kB
Inactive:         979432 kB
Active(anon):    6173036 kB
Inactive(anon):   927932 kB
Active(file):      51196 kB
Inactive(file):    51500 kB
...
Shmem:             10000 kB
...
SReclaimable:      43532 kB
...
```

> page cache = Buffers + Cached + SwapCached = Active(file) + Inactive(file) + Shmem + SwapCached

