---
title: JVM 工具
date: 2023-04-06 13:45:43
categories:
  - java
tags:
  - jvm
---

# 定位 cpu 占用高的线程

1. jps 获取 Java 进程的 PID。
1. jstack -l PID >> thread_stack.txt 导出 CPU 占用高进程的线程栈
1. top -H -p PID 查看对应进程的哪个线程占用 CPU 过高。
1. printf '%x\n' TID。转换为 16 进制。
1. 在导出的 thread_stack.txt 中查找转换成为16进制的线程 PID。找到对应的线程栈。

> 使用 jstack 在捕获线程快照时，JVM 会短暂挂起所有线程以确保线程栈的状态一致，但这个挂起时间通常非常短，几乎可以忽略影响。



# 导出 jvm 内存

```java
jmap -dump:live,format=b,file=java_heap.dump pid
```

> jmap -dump 会导致 JVM 挂起所有线程，当内存较大时，会有明显卡顿。



# 查看 GC 信息

```java
jstat -gcutil pid [interval]ms [times]
```

参数解释

```
S0C：第一个幸存者区（Survivor Space）容量

S1C：第二个幸存者区（Survivor Space）容量

S0U：第二个幸存者区使用量

S1U：第二个幸存者区使用量

EC：伊甸园去容量

EU：伊甸园区使用量

OC：Old Generation区容量

OU：Old Generation使用量

MC：Mataspace区容量

MU：Mataspace区实际使用量

CCSC：压缩类空间大小（不是很懂，先标记一下）

CCSU：压缩类空间使用率（不是很懂，先标记一下）

YGC：年轻代垃圾回收次数

YGCT：年轻代垃圾回收时间

FGC：年老代垃圾回收次数

FGCT：年老代垃圾回收时间

GCT：总垃圾回收时间

S0：S0C区域使用率

S0：S1C区域使用率

E：伊甸园去使用率

O：Old Generation使用率，OU/OC

M：Matespace区使用率，MU/MC

CCS：压缩类空间使用率
```



# 其他参数

- 查看运行参数： `jinfo -flags pid`。
- 查看 JVM 堆内存使用情况：`jmap -heap pid`。
- 手动触发 GC：`jcmd <pid> GC.run`。

