---
title: Arthas运行原理
date: 2020-03-08 23:09:56
tags:
 - Arthas
 - Java
 - JVM
categories:
 - Java
---

[Arthas](https://github.com/alibaba/arthas)是一款Java诊断利器，之前用它定位过不少问题，也在团队内部做过分享。那么Arthas是如何attach到Jvm，又是如何实现watch、trace命令和生成火焰图，一步步成为巫妖王的呢？

<!-- more -->

# Instrument和Attach API

Jdk5增加了一个包java.lang.instrument，提供了对Jvm底层组件的访问能力，Instrument要求在运行前利用命令行参数或者系统参数设置代理类，VM启动完成之后（绝大多数类加载前）初始化。

开发基于instrument的应用，需要这么几个步骤：

1. 编写premain函数
2. jar文件打包，制定Premain-Class
3. 使用-javaagent参数启动

Jdk6以后，针对这点进行了改进，开发者可以在main函数执行之后再启动自己的Instrument应用，入口是agentmain函数。arthas就是通过这个实现的。

之后就可以通过addTransformer，retransformClasses，redefineClasses等方式对字节码进行增强和热替换了。

# ASM

ASM是一个Java字节码操作框架，用来动态生成class或者增强class，cglib的底层就是它，arthas也是通过它实现对class的增强的。

Arthas增强功能的核心是Enhancer和AdviceWeaver这两个类，对方法进行Aop织入，达到watch，trace等效果。

这里以watch命令为例：

![watch命令](http://media.kosho.tech/blog/20200321/arthas-watch.png)

AdviceWeaver#onMethodEnter

```java
protected void onMethodEnter() {
    codeLockForTracing.lock(new CodeLock.Block() {
        @Override
        public void code() {
            final StringBuilder append = new StringBuilder();
            
            // 加载before方法
            loadAdviceMethod(KEY_ARTHAS_ADVICE_BEFORE_METHOD);

            // 推入Method.invoke()的第一个参数
            pushNull();

            // 方法参数
            loadArrayForBefore();

            // 调用方法
            invokeVirtual(ASM_TYPE_METHOD, ASM_METHOD_METHOD_INVOKE);
            pop();
        }
    });

    mark(beginLabel);
}
```

WatchAdviceListener

```java
    public void before(ClassLoader loader, Class<?> clazz, ArthasMethod method, Object target, Object[] args)
            throws Throwable {
        // 开始计算本次方法调用耗时
        threadLocalWatch.start();
        if (command.isBefore()) {
            watching(Advice.newForBefore(loader, clazz, method, target, args));
        }
    }

    public void afterReturning(ClassLoader loader, Class<?> clazz, ArthasMethod method, Object target, Object[] args,
                               Object returnObject) throws Throwable {
        Advice advice = Advice.newForAfterRetuning(loader, clazz, method, target, args, returnObject);
        if (command.isSuccess()) {
            watching(advice);
        }

        finishing(advice);
    }

    private void watching(Advice advice) {
        try {
            // 本次调用的耗时
            double cost = threadLocalWatch.costInMillis();
            if (isConditionMet(command.getConditionExpress(), advice, cost)) {
                Object value = getExpressionResult(command.getExpress(), advice, cost);
                String result = StringUtils.objectToString(
                        isNeedExpand() ? new ObjectView(value, command.getExpand(), command.getSizeLimit()).draw() : value);
                process.write("ts=" + DateUtils.getCurrentDate() + "; [cost=" + cost + "ms] result=" + result + "\n");
                process.times().incrementAndGet();
                if (isLimitExceeded(command.getNumberOfLimit(), process.times().get())) {
                    abortProcess(process, command.getNumberOfLimit());
                }
            }
        } catch (Exception e) {
            logger.warn("watch failed.", e);
            process.write("watch failed, condition is: " + command.getConditionExpress() + ", express is: "
                          + command.getExpress() + ", " + e.getMessage() + ", visit " + LogUtil.LOGGER_FILE
                          + " for more details.\n");
            process.end();
        }
    }
```

# JVMTI

然后呢？还没有结束，继续看一下attach下面一层是什么。

JVMTI（JVM Tool Interface）是Java虚拟机所提供的native接口，提供了可用于debug和profiler的能力，是实现调试器和其他运行态分析工具的基础，Instrument就是对它的封装。

JVMTI又是在JPDA（Java Platform Debugger Architecture）之下的三层架构之一，JVMTI，JDWP，JDI。可以参考IBM系列文章：[深入 Java 调试体系](https://www.ibm.com/developerworks/cn/java/j-lo-jpda1/index.html?ca=drs-)

我们常用的debug参数-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8080，其实就是加载了jdwp的lib，开启了调试端口，然后就可以通过JDI接口进行交互调试。

看下java8里attach的入口源码：LinuxVirtualMachine.java

```java
/**
 * Attaches to the target VM
 */
LinuxVirtualMachine(AttachProvider provider, String vmid)
    throws AttachNotSupportedException, IOException {
    super(provider, vmid);
   
    // This provider only understands pids
    int pid;
    try {
        pid = Integer.parseInt(vmid);
    } catch (NumberFormatException x) {
        throw new AttachNotSupportedException("Invalid process identifier");
    }
   
    // Find the socket file. If not found then we attempt to start the
    // attach mechanism in the target VM by sending it a QUIT signal.
    // Then we attempt to find the socket file again.
    path = findSocketFile(pid);
    if (path == null) {
        File f = createAttachFile(pid);
        try {
            // On LinuxThreads each thread is a process and we don't have the
            // pid of the VMThread which has SIGQUIT unblocked. To workaround
            // this we get the pid of the "manager thread" that is created
            // by the first call to pthread_create. This is parent of all
            // threads (except the initial thread).
            if (isLinuxThreads) {
                int mpid;
                try {
                    mpid = getLinuxThreadsManager(pid);
                } catch (IOException x) {
                    throw new AttachNotSupportedException(x.getMessage());
                }
                assert(mpid >= 1);
                sendQuitToChildrenOf(mpid);
            } else {
                sendQuitTo(pid);
            }
   
            // give the target VM time to start the attach mechanism
            int i = 0;
            long delay = 200;
            int retries = (int)(attachTimeout() / delay);
            do {
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException x) { }
                path = findSocketFile(pid);
                i++;
              } while (i <= retries && path == null);
              if (path == null) {
                  throw new AttachNotSupportedException(
                      "Unable to open socket file: target process not responding " +
                      "or HotSpot VM not loaded");
              }
          } finally {
              f.delete();
          }
      }
     
      // Check that the file owner/permission to avoid attaching to
      // bogus process
      checkPermissions(path);
     
      // Check that we can connect to the process
      // - this ensures we throw the permission denied error now rather than
      // later when we attempt to enqueue a command.
      int s = socket();
      try {
          connect(s, path);
      } finally {
          close(s);
      }
  }
```

Linux上，动态attach大概分为这么几步：

- 加载libattach.so动态链接库
- 创建attach_pid文件
- 发送SIGQUIT信号
- JVM创建一个路径为/tmp/.java_pid的UNIX Socket
- 建立连接

然后就可以通过如下的通讯格式进行交互了：

```xml
<PROTOCAL VERSION>\0<COMMAND>\0<ARG1>\0<ARG2>\0<ARG3>
```
这个机制和上面代码也说明了为什么不能远程attach，以及不同用户之间attach为什么会失败。

-agentlib:agent-lib-name=options和-agentpath:path-to-agent=options这两种方式则会调用这么一个JNI的方法：

JNIEXPORT jint JNICALL Agent_Onload(JavaVM* vm, char *options, void *reserved)

会直接传递VM指针进来，就可以拿到JVMTI指针，相比较java代码功能会更强大一些。

# 火焰图原理（async-profiler）

Arthas的功能其实到上面已经结束啦，但是arthas还集成了另一个大杀器，async-profiler（IDEA的 JVM Profiler也是集成的这个），可以直接对java热点方法生成火焰图！

通常Arthas的trace命令用来定位单点性能问题，但是如果系统整体启动、运行都很慢，那Arthas也力不从心了，需要对系统全局做性能热点分析和优化，这个时候火焰图就派上了用场。

对于大部分开源Profiler，原理其实就是循环打印线程堆栈，然后统计各方法出现的频率，比如说Jprofiler的Simpling模式（Instrumentation模式是对所有类进行增强，比较精准一些，但是性能影响较大），网上也有一些shell工具，可以打印堆栈生成数据表，导入Excel用透视图表进行分析。

这些方式有一个问题，就是JVM只会在安全点（safe point）进行采样，如果某些方法执行时间极短，但是频率很高，实际占用了大量的cpu time，但是采样周期不能无限调小，导致大量的样本调用堆栈并不存在这些高频小方法，导致最终统计结果无法反应真实的cpu热点。

> JVM安全点如何定义和划分，R大的回答：https://www.zhihu.com/question/29268019/answer/43762165

Async profiler则是通过JVMTI + AsyncGetCallTrace（非JVM标准函数）实现的，基本上可以采样任何时期的stack状态，Linux环境下结合perf_event还能做到同时采样Java栈和Native栈，同时分析Native代码的热点。

生成的调用堆栈形式如下，左边是调用栈方法排列，右边是样本出现次数：

base_func;func1;func2;func3 10

base_func;funca;funcb 15

然后通过FlameGraph等工具处理，就可以生成火焰图了。

![火焰图Demo](https://alibaba.github.io/arthas/_images/arthas-output-svg.jpg)