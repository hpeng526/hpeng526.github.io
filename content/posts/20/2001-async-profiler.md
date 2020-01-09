---
title: async-profiler
date: 2020-01-08 23:00:00
tags: [java, profiler]
---

# async-profiler

一个低开销的, 不会有安全点偏差问题(Safepoint bias problem)的 Java 采样分析器. 它使用 HotSpot-specific APIs (AsyncGetCallTrace) 来收集堆栈和内存分配信息, perf_events 收集系统调用. 可以运行在任何基于 HotSpot JVM 上虚拟机, 例如OpenJDK, Oracle JDK.

## 支持类型

Async Profiler 支持一下输出类型

- summary
- traces
- flat
- collapsed
- svg
- tree
- jfr

其中 svg, tree, jfr 需要有个文件输出

输出文件后缀会自动确定输出格式 *.svg, *.html *.jfr 对应 svg, tree, jfr

summary, traces, flat 可以组合 `summary,traces=200,flat=200`

## 常用命令

Linux 4.6 以上, 非root用户要采集内核调用栈需要修改两个值

``` sh
# echo 1 > /proc/sys/kernel/perf_event_paranoid
# echo 0 > /proc/sys/kernel/kptr_restrict
```

火焰图,纵向从下至上,代表方法间的调用关系,横条的宽度代表所占CPU时间的多少.

``` sh
# -d 采集时间为60s
# -e itimer 内核参数不符合, 没法变更, 此参数为降级参数, 不读取 perf_event, 只采集java
# -f 输出到 res.svg 文件, 根据后缀名是svg, 输出火焰图
./profiler.sh -e itimer -d 60 -f res.svg pid
```

### 参考

[https://github.com/jvm-profiling-tools/async-profiler](https://github.com/jvm-profiling-tools/async-profiler)
