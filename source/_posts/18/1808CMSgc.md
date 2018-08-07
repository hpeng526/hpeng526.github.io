---
title: 从 GC log 了解与分析 gc
date: 2018-08-06 22:01:00
tags: [JVM, CMS, GC]
---

## CMS 介绍

CMS, Concurrent Mark Sweep, 并发的标记清楚算法GC.

### CMS 执行

1. 初始化标记
1. 并发标记
1. 并发预清理
1. 重标记
1. 并发清理
1. 重置

环境配置是 JDK8, HotSpot VM

应用配置的参数
```
-XX:+UseConcMarkSweepGC                             // 使用 CMS GC
-XX:+UseCMSInitiatingOccupancyOnly                  // 不基于运行时收集的数据来启动 CMS 垃圾收集的周期, 使用CMSInitiatingOccupancyFraction
-XX:CMSInitiatingOccupancyFraction=70               // 使用70%后开始 CMS GC
-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses   // 显式 GC 执行 CMS GC 非 FGC
-XX:+CMSClassUnloadingEnabled                       // CMS 也收集 PermGen space (https://blogs.oracle.com/poonam/about-g1-garbage-collector,-permanent-generation-and-metaspace)
-XX:+ParallelRefProcEnabled                         // 并行处理 ref
-XX:+CMSScavengeBeforeRemark                        // 在CMS remark前执行一次minor gc清理新生代, 减少stop-the-world时间
-XX:+PrintGCDetails                                 // 打印 GC 详细信息
-XX:+PrintGCDateStamps                              // 输出GC的时间戳
```

<!-- more -->

CMS GC log 如下
``` CMS GC log
// initial mark stop-the-world
2018-08-06T10:55:07.447+0800: 495808.570: [GC (CMS Initial Mark) [1 CMS-initial-mark: 1421882K(1966080K)] 1464386K(3027776K), 0.0610635 secs] [Times: user=0.05 sys=0.00, real=0.06 secs] 
// concurrent mark
2018-08-06T10:55:07.510+0800: 495808.633: [CMS-concurrent-mark-start]
2018-08-06T10:55:38.879+0800: 495840.002: [CMS-concurrent-mark: 27.341/31.368 secs] [Times: user=48.82 sys=8.29, real=31.36 secs] 
// concurrent preclean
2018-08-06T10:55:38.879+0800: 495840.002: [CMS-concurrent-preclean-start]
2018-08-06T10:55:39.303+0800: 495840.426: [CMS-concurrent-preclean: 0.131/0.424 secs] [Times: user=0.89 sys=0.05, real=0.43 secs] 
// abortable preclean
2018-08-06T10:55:39.303+0800: 495840.427: [CMS-concurrent-abortable-preclean-start]
2018-08-06T10:55:39.303+0800: 495840.427: [CMS-concurrent-abortable-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
// final remark stop-the-world
2018-08-06T10:55:39.320+0800: 495840.444: [GC (CMS Final Remark) [YG occupancy: 237032 K (1061696 K)]2018-08-06T10:55:39.321+0800: 495840.444: [GC (CMS Final Remark) 2018-08-06T10:55:39.321+0800: 495840.444: [ParNew: 237032K->237032K(1061696K), 0.0000520 secs] 2068900K->2068900K(3027776K), 0.0004108 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
// rescan
2018-08-06T10:55:39.321+0800: 495840.444: [Rescan (parallel) , 0.1066672 secs]2018-08-06T10:55:39.428+0800: 495840.551: [weak refs processing, 0.0868804 secs]2018-08-06T10:55:39.515+0800: 495840.638: [class unloading, 0.0667963 secs]
// scrub symbol table
2018-08-06T10:55:39.582+0800: 495840.705: [scrub symbol table, 0.0554765 secs]2018-08-06T10:55:39.637+0800: 495840.761: [scrub string table, 0.0065050 secs][1 CMS-remark: 1831867K(1966080K)] 2068900K(3027776K), 0.3247966 secs] [Times: user=0.48 sys=0.01, real=0.32 secs] 
2018-08-06T10:55:39.646+0800: 495840.770: [CMS-concurrent-sweep-start]
2018-08-06T10:55:48.631+0800: 495849.755: [CMS-concurrent-sweep: 6.052/8.985 secs] [Times: user=7.66 sys=4.13, real=8.99 secs] 
// reset start
2018-08-06T10:55:49.130+0800: 495850.253: [CMS-concurrent-reset-start]
2018-08-06T10:55:49.139+0800: 495850.263: [CMS-concurrent-reset: 0.009/0.009 secs] [Times: user=0.03 sys=0.00, real=0.00 secs] 
2018-08-06T10:55:53.548+0800: 495854.672: [CMS: 1336369K->468683K(1966080K), 2.2096604 secs] 2336499K->468683K(3027776K), [Metaspace: 120912K->120912K(1165312K)], 2.7556633 secs] [Times: user=2.87 sys=0.01, real=2.75 secs] 

```

## 知识点

### CMS 阶段

步骤 | 解释 | stop-the-world
---------|----------|---------
 CMS Initial Mark | 可达性分析, 标记 GC Roots `直接关联` 对象 | True
 CMS-concurrent-mark | GC Tracing, 从 GC Roots 开始对堆进行`可达性分析`, 找出存活对象| False
 CMS-concurrent-preclean | 负责前一个阶段标记了又发生改变的对象标记 | False
 CMS-concurrent-abortable-preclean | 尝试着去承担 Final Remark 阶段足够多的工作, 重复的做相同的事情直到 abort 条件 | False 
 CMS Final Remark | 完成标记整个年老代的所有的存活对象 | True
 CMS-concurrent-sweep | 开始移除对象 | False
 CMS-concurrent-reset | 重置 CMS 算法内部结构 | False

### GC 触发条件

原因 | 说明 
---------|----------
 Allocation Failure | 新生代(Young generation)不够空间给创建新对象, 触发 minor gc.
 Promotion Failure | 没有 `连续` 的空间来存放更大的对象. 通常会导致 FGC.
 GCLocker Initiated GC | JNI 临界区被释放就会触发
 Concurrent Mode Failure | CMS 采用多个线程和应用线程并发执行, 减少停顿时间, 通常发生于年老代被用完之前不能完成对无引用对象的回收或者年老代不能满足新的空间分配
 Heap Dump Initiated GC | heap dump 之前进行 FGC , 一般是通过工具 dump 会触发 (jmap)

### GC 异常点

通过 [gceasy.io](gceasy.io) 上传日志, 分析异常点得出

`It looks like your application is waiting due to lack of compute resources (either CPU or I/O cycles). Serious production applications shouldn't be stranded because of compute resources. In 47 GC event(s), 'real' time took more than 'usr' + 'sys' time.`

Timestamp | User Time (secs) | Sys Time (secs) | Real Time (secs)
---------|----------|---------|--------
2018-08-06T10:21:12 | 0.26 | 0.01 | 0.61
2018-08-06T10:21:18 | 0.24 | 0.04 | 0.9
2018-08-06T10:33:25 | 0.22 | 0.01 | 0.58
2018-08-06T10:34:54 | 0.3 | 0.01 | 0.52
2018-08-06T10:35:15 | 0.55 | 0.02 | 1.15

意思就是 'real' time > 'usr' + 'sys', 造成这种现象的可能性有

1. Heavy I/O 
如果有 I/O 密集任务, 'real time' 会变长.
    > 解决方案
        1. 优化高 I/O 任务
        1. 在服务器上减少导致高 I/O 的任务
        1. 迁移应用到低 I/O 的服务器上
2. Lack of CPU
如果是服务器上有多个进程在跑, 程序得不到足够的 CPU 周期, 就会开始等待. 因此 'real time'会变长
    > 解决方案:
        1. 减少服务器上的进程
        1. 增加 CPU
        1. 迁移应用到充足 CPU 服务器上

### 现象分析

当时出现的现象是:
1. CPU 告警
1. 上去服务器看到系统负载已经到70左右
1. top 信息看到应用占用 CPU 高
1. GC 日志显示同一台服务器上的两个服务在10点时间段 CMS 都在工作
1. 业务高峰期
1. 4核服务器

初步分析: 
1. 从应用线程dump分析, 应用内线程无明显异常现象. 
1. 结合 top -H 初步得到是 GC 线程在运行
1. 从 GC log 入手, 分析得出异常点, 回顾与计算两个服务 CMS 线程占用了一半的 CPU 资源 `(4+3)/4*2`, 导致 CPU 资源紧缺, 整体服务变慢.



### 参考资料

1. [https://www.cnblogs.com/zhangxiaoguang/p/5792468.html](https://www.cnblogs.com/zhangxiaoguang/p/5792468.html)
1. [https://juejin.im/post/5b651200f265da0fa00a38d7?utm_source=gold_browser_extension](https://juejin.im/post/5b651200f265da0fa00a38d7?utm_source=gold_browser_extension)
1. [https://stackoverflow.com/questions/22149075/why-promotion-failure-and-concurrent-mode-failure](https://stackoverflow.com/questions/22149075/why-promotion-failure-and-concurrent-mode-failure)
1. [https://blog.gceasy.io/2016/12/08/real-time-greater-than-user-and-sys-time/](https://blog.gceasy.io/2016/12/08/real-time-greater-than-user-and-sys-time/)