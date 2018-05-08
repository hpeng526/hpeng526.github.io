---
title: btrace 笔记
date: 2018-04-28 22:10:00
tags: [travis, hexo]
---

[Btrace](https://github.com/btraceio/btrace) - a safe, dynamic tracing tool for the Java platform

# Brtace 使用

``` sh
btrace {pid} Debug.java
```

<!-- more -->

## 方法内方法调用

``` java

@OnMethod(clazz = "/.*className.*/", method = "/.*methodName.*/",
        location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/", where = Where.AFTER))
    public static void methodCall(@ProbeClassName String probeClassName,
                                   @ProbeMethodName String probeMethod,
                                   @TargetInstance Object tInstance,
                                   @TargetMethodOrField String tMethod,
                                   @Duration long time) {
        if (time / 100000 > 0) {
            BTraceUtils.println(probeClassName + "#" + probeMethod + " \n\t " + tInstance + "#" + tMethod);
            BTraceUtils.println("cost: " + time / 100000);
        }
    }

```

## 方法调用盏

``` java
@OnMethod(clazz = "/.*ProductService.*/", method = "/.*fillProduct.*/",
        location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/", where = Where.AFTER))
    public static void callStack(@ProbeClassName String probeClassName,
                                 @ProbeMethodName String probeMethod) {
        BTraceUtils.println("----------------" + probeClassName + "#" + probeMethod);
        BTraceUtils.jstack();
    }
```



