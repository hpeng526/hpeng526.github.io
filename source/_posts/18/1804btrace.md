---
title: btrace 笔记
date: 2018-04-28 22:10:00
tags: [travis, hexo]
---

[Btrace](https://github.com/btraceio/btrace) - a safe, dynamic tracing tool for the Java platform

# Btrace 加载过程

## client

- Javac 加载 Btrace Verifier
- 编译成字节码
- 通过 attach 方法动态 attach 到 JVM [Attach API](https://docs.oracle.com/javase/7/docs/jdk/api/attach/spec/com/sun/tools/attach/spi/AttachProvider.html)
- submit 提交 Btrace 编译后的字节码到对应 PID VM.(socket 通信)

```java
try {
    Client client = new Client(port, OUTPUT_FILE, PROBE_DESC_PATH,
        DEBUG, TRACK_RETRANSFORM, TRUSTED, DUMP_CLASSES, DUMP_DIR, statsdDef);
    if (! new File(fileName).exists()) {
        errorExit("File not found: " + fileName, 1);
    }
    byte[] code = client.compile(fileName, classPath, includePath);
    if (code == null) {
        errorExit("BTrace compilation failed", 1);
    }
    client.attach(pid, null, classPath);
    registerExitHook(client);
    if (con != null) {
        registerSignalHandler(client);
    }
    if (isDebug()) debugPrint("submitting the BTrace program");
    client.submit(fileName, code, btraceArgs,
        createCommandListener(client));
} catch (IOException exp) {
    errorExit(exp.getMessage(), 1);
}
```

<!-- more -->

``` java
/**
 * Attach the BTrace client to the given Java process.
 * Loads BTrace agent on the target process if not loaded
 * already.
 */
public void attach(String pid, String sysCp, String bootCp) throws IOException {
    try {
        String agentPath = "/btrace-agent.jar";
        String tmp = Client.class.getClassLoader().getResource("com/sun/btrace").toString();
        tmp = tmp.substring(0, tmp.indexOf('!'));
        tmp = tmp.substring("jar:".length(), tmp.lastIndexOf('/'));
        agentPath = tmp + agentPath;
        agentPath = new File(new URI(agentPath)).getAbsolutePath();
        attach(pid, agentPath, sysCp, bootCp);
    } catch (RuntimeException re) {
        throw re;
    } catch (IOException ioexp) {
        throw ioexp;
    } catch (Exception exp) {
        throw new IOException(exp.getMessage());
    }
}
```

## agent // TODO

- 添加 classpath
- 开启一个 ServerSocket
- 收到指令后, 主要有
  - 重写类 遍历当前class, 正则匹配到class
  - 转换类 替换原来的class

`processClasspaths(libs);`

```java

// ignore String bootClassPath = argMap.get("bootClassPath");
nst.appendToBootstrapClassLoaderSearch(jf);
// ignore

// ignore String systemClassPath = argMap.get("systemClassPath");
inst.appendToSystemClassLoaderSearch(jf);
// ignore

ddPreconfLibs(libs);

```

```java
private static synchronized void main(final String args, final Instrumentation inst) {
    if (Main.inst != null) {
        return;
    } else {
        Main.inst = inst;
    }

    try {
        loadArgs(args);
        // set the debug level based on cmdline config
        settings.setDebug(Boolean.parseBoolean(argMap.get("debug")));
        if (isDebug()) {
            debugPrint("parsed command line arguments");
        }
        parseArgs();

        int startedScripts = startScripts();

        String tmp = argMap.get("noServer");
        // noServer is defaulting to true if startup scripts are defined
        boolean noServer = tmp != null ? Boolean.parseBoolean(tmp) : hasScripts();
        if (noServer) {
            if (isDebug()) {
                debugPrint("noServer is true, server not started");
            }
            return;
        }
        Thread agentThread = new Thread(new Runnable() {
            @Override
            public void run() {
                BTraceRuntime.enter();
                try {
                    startServer();
                } finally {
                    BTraceRuntime.leave();
                }
            }
        });
        BTraceRuntime.initUnsafe();
        BTraceRuntime.enter();
        try {
            agentThread.setDaemon(true);
            if (isDebug()) {
                debugPrint("starting agent thread");
            }
            agentThread.start();
        } finally {
            BTraceRuntime.leave();
        }
    } finally {
        inst.addTransformer(transformer, true);
        Main.debugPrint("Agent init took: " + (System.nanoTime() - ts) + "ns");
    }
}
```

``` java
void retransformLoaded() throws UnmodifiableClassException {
    if (runtime != null) {
        if (probe.isTransforming() && settings.isRetransformStartup()) {
            ArrayList<Class> list = new ArrayList<>();
            debugPrint("retransforming loaded classes");
            debugPrint("filtering loaded classes");
            ClassCache cc = ClassCache.getInstance();
            for (Class c : inst.getAllLoadedClasses()) {
                if (c != null) {
                    cc.get(c);
                    if (inst.isModifiableClass(c) &&  isCandidate(c)) {
                        debugPrint("candidate " + c + " added");
                        list.add(c);
                    }
                }
            }
            list.trimToSize();
            int size = list.size();
            if (size > 0) {
                Class[] classes = new Class[size];
                list.toArray(classes);
                startRetransformClasses(size);
                if (isDebug()) {
                    for(Class c : classes) {
                        try {
                            debugPrint("Attempting to retransform class: " + c.getName());
                            inst.retransformClasses(c);
                        } catch (VerifyError e) {
                            debugPrint("verification error: " + c.getName());
                        }
                    }
                } else {
                    inst.retransformClasses(classes);
                }
            }
        }
        runtime.send(new OkayCommand());
    }
}
```

``` java

@Override
public synchronized byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
    if (probes.isEmpty()) return null;

    className = className != null ? className : "<anonymous>";

    if ((loader == null || loader.equals(ClassLoader.getSystemClassLoader())) && isSensitiveClass(className)) {
        if (isDebug()) {
            debugPrint("skipping transform for BTrace class " + className); // NOI18N
        }
        return null;
    }

    if (filter.matchClass(className) == Filter.Result.FALSE) return null;

    boolean entered = BTraceRuntime.enter();
    try {
        if (isDebug()) {
            debug.dumpClass(className.replace('.', '/') + "_orig", classfileBuffer);
        }
        BTraceClassReader cr = InstrumentUtils.newClassReader(loader, classfileBuffer);
        BTraceClassWriter cw = InstrumentUtils.newClassWriter(cr);
        for(BTraceProbe p : probes) {
            p.notifyTransform(className);
            cw.addInstrumentor(p, loader);
        }
        byte[] transformed = cw.instrument();
        if (transformed == null) {
            // no instrumentation necessary
            if (isDebug()) {
                debugPrint("skipping class " + cr.getJavaClassName());
            }
            return classfileBuffer;
        } else {
            if (isDebug()) {
                debugPrint("transformed class " + cr.getJavaClassName());
            }
            if (debug.isDumpClasses()) {
                debug.dumpClass(className.replace('.', '/'), transformed);
            }
        }
        return transformed;
    } catch (Throwable th) {
        debugPrint(th);
        throw th;
    } finally {
        if (entered) {
            BTraceRuntime.leave();
        }
    }
}

```

# Brtace 使用

``` sh
btrace {pid} Debug.java
```

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
@OnMethod(clazz = "/.*className.*/", method = "/.*methodName.*/",
        location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/", where = Where.AFTER))
    public static void callStack(@ProbeClassName String probeClassName,
                                 @ProbeMethodName String probeMethod) {
        BTraceUtils.println("----------------" + probeClassName + "#" + probeMethod);
        BTraceUtils.jstack();
    }
```



