# 关于现代GC ZGC那茬事

## 简介

简单了解下新的垃圾收集器的历程：

G1（java 9 默认 oracle ） >  ZGC（JDK 11之后 openjdk ） >  shenandoah（JDK 12 发布，兼容jdk 8 11, 15 openjdk ) 

这几个垃圾收集器实现都是分为多个Region的 (g1 分代，zgc，shenandoah 未分代，但是ZGC后续会实现分代收集的目标），针对的是大内存多核的平台。

**GC 优化的两个两个目标**

1. 吞吐量 (GC线程会与用户线程争夺CPU时钟周期,所以吞吐量指非GC的时执行任务的时间，用户线程执行时间 + GC线程执行时间 = 程序时间)
2. 暂停时间 (简称STW， GC执行相关任务如标记垃圾，迁移对象,期间用户线程暂停无进展) 

ZGC出现之前,两者很难兼顾。吞吐量一般可以通过调整堆的大小来解决，增加吞吐量就得少运行GC，也意味着单次GC前要做更多的事，导致暂停时间增加，大堆中暂停时间增加尤为明显。

## ZGC

### ZGC工作流程图![](..\images\zgc-1.png)

可以看到只有红色箭头出才会造成STW, 居官方说过一次GC暂停时间<10ms。

![一个简单的对比图](..\images\zgc-0.png)

​																一个简单的对比图，大堆下的GC STW时间

还等什么？直接 java -XX:UseZGC 起飞！

### How to do that？

**1 有色指针(colored pointers)**

![](..\images\zgc-2.png)

一个对象的指针包括16位的元数据(锁标志，类元信息之类的) + 4位有色指针 + 44位的对象地址(所以ZGC可以支持2的44次方约16TB的大堆,但这也带来了更大的内存消耗，因为指针不再支持压缩)

**2 加载屏障 （load barrier)**

JIT编译器会在从堆中加载对象时加入相应的策略，及确认对象指针上的是否有bad color，来确实是否是垃圾对象，并进行相应的处理。具体步骤如下

![](..\images\zgc-3.png)

![](..\images\zgc-4.png)

### 关于ZGC优化

ZGC优化相较于其他的垃圾收集器简单了多了，调优的参数有以下几个：

1.  -Xmx<size> 最大堆大小 如之前所述支持最大到16TB的堆，而且ZGC并不会随堆的增大而增加暂停时间，但是并不意味着你要浪费内存

2.  -XX:ConcGCThreads=<numbers>  并发GC线程，建议不超过CPU 70%的负载线程数

3.  启用 hugepages   需要配置实际给zgc使用的更多JVM还要进行其它操作及维护内部数据结构

   e.g  16g的堆 需要分配 16g / 2mb = 8192 个 hugepages , 所以准备了 19g  9216个 hugepages

   ```shell
   echo ``9216` `> /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
   ```

4.  -XX:+UseNUMA  在NUMA计算机（例如多插槽x86计算机）上运行时，启用NUMA支持通常可以显着提高性能。

 [更多的参数请参照ZGC WIKI](https://wiki.openjdk.java.net/display/zgc/Main#Main-Configuration&Tuning)

### 启用GC日志记录

使用以下命令行选项启用GC日志记录：

```
-Xlog:<tag set>,[<tag set>, ...]:<log file>
```

有关此选项的一般信息/帮助，请执行以下操作：

```
-Xlog:help
```

要启用基本日志记录（每个GC一行输出）：

```
-Xlog:gc:gc.log
```

要启用对调优/性能分析有用的GC日志记录，请执行以下操作：

```
-Xlog:gc*:gc.log
```

其中`gc*`表示记录包含该`gc`标记的所有标记组合，并且`:gc.log`表示将该日志写入名为的文件`gc.log`。



## 结尾

目前ZGC还是并没有分代，却已获得如此**战绩**，实现分代后性能可想而知（已经在进行中，ZGC的最新进展可以关注下 [zgc项目leader](https://malloc.se/)的博客与 [ZGC ](https://github.com/openjdk/jdk/tree/master/src/hotspot/share/gc/z)储库）



## 参考文献：

[ZGC-OracleDevLive-2020.pdf](../pdf/ZGC-OracleDevLive-2020.pdf)

[ZGC WIKI](https://wiki.openjdk.java.net/display/zgc/Main)



