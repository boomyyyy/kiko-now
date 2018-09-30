---
layout: post
title: "JVM 故障处理常用命令"
date: 2018-09-02
tags: [Java, JVM, 工具]
comments: true
share: true
---

## 概述
前面几篇文章介绍了 JVM 的内存分配机制和垃圾回收机制，这篇文章主要介绍一些在当 JVM 发生故障时定位分析问题所使用到的一些工具，使用这些工具可以用来打印运行日志、异常堆栈、GC 日志 或者导出线程快照（threaddump/javacore 文件）、堆转储快照（heapdump/hprof 文件）等。常用的工具如下:

|名称|主要作用|
|---|---|
|jps|JVM Process Status Tool，显示指定系统内所有的 HotSpot 虚拟机进程|
|jstat|JVM Statictics Montoring Tool，用于收集HotSpot 虚拟机各方面的运行数据|
|jinfo|Configuration Info for Java，显示虚拟机配置信息|
|jmap|Memory Map for Java，生成虚拟机的内存转储快照（heapdump 文件）|
|jhat|Jvm Heap Dump Browser，用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以再浏览器上可以查看分析结果|
|jstack|Stack Trace for Java，显示虚拟机进程快照|


## jps
jps(JVM Process Status)可以列出正在运行的虚拟机进程，冰系那是虚拟机执行主类（Main Class，main() 函数所在的类)）名称以及这些进程的本地虚拟机唯一 ID（Local Virtual Machine Identifier，LVMID）。该命令是使用频率最高的 JDK 命令工具，因为其他的 JDK 工具大多数需要输入它查询到的 LVMID 来确定要监控的是哪一个虚拟机进程。对于本地虚拟机进程来说，LVMID 与操作系统的进程 ID （Process Identifier， PID）是一致的。

### 命令格式
```bash
jps [options] [hostid]
```
hostid 可以通过 RMI 协议查询开启了 RMI 服务的远程虚拟机进程状态，hostid 为 RMI 注册表中的主机名。

### 执行样例
```bash
➜  ~ jps -l
47633 
73929 sun.tools.jps.Jps
47833 org.jetbrains.idea.maven.server.RemoteMavenServer
48044 org.jetbrains.jps.cmdline.Launcher
46525 org.codehaus.plexus.classworlds.launcher.Launcher
5838 org.codehaus.plexus.classworlds.launcher.Launcher
➜  ~
```

### 主要选项

|选项|作用|
|---|---|
|-q|只输出 LVMID，省略主类的名称|
|-m|输出虚拟机进程启动时传递给主类 main() 函数的参数|
|-l|输出主类的全名，如果进程执行的是 jar 包，输出 jar 路径|
|-v|输出虚拟机进程启动时的 JVM 参数|


***命令相关选项及打印信息可参考 [jps](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html#CHDCGECD){:target="_blank"}***

## jstat
jstat（JVM Statistics Monitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具。可以本地活着远程虚拟机进程的类装载、内存、垃圾收集、JIT 编译等运行数据。

### 命令格式
1. 本地虚拟机进程
```bash
jstat [ option vmid [interval[s|ms] [count]] ]
```

2. 远程虚拟机进程
```bash
jstat [ option [protocol:][//]lvmid[@hostname[:port]/servername] [interval[s|ms] [count]] ]
```
 
> 参数 interval 和 count 表示查询间隔和次数，如果省略这两个参数，则代表只查询一次。其中  `jstat -gcutil 2764 250 20` 则表示每隔 250 毫秒查询一次进程 2764 的垃圾收集情况，一共查询 20 次。 option 代表着用户希望查询的虚拟机信息，主要分为三类：类装载、垃圾收集、运行期编译状况。

### 执行样例
```bash
➜  ~ jstat -gcutil 5838
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
  0.00   0.00  63.67  19.77  97.57  93.01    4    0.053     1    0.060    0.113
➜  ~ 
```

> 其中 E 表示 Eden 区使用了 63.67% 的空间，两个 Survivor 区（S0、S1，表示 Survivor0、Survivor1）里面都是空的），O 表示 Old 及老年代和 M 表示 Mataspace 方法区分别使用了 19.77% 和 97.57% 的空间。CCS 表示压缩泪空使用了 93.01（这项信息）。O 表示 Old 即老年代使用了 19.77% 的空间，M 表示 mataspace 即元数据区和 CCS 表示 matespace 中的类压缩空间分别使用了 97.57% 和 93.01%。程序运行以来共发生 Minior GC(YGC 表示 Young GC) 4 次。总耗时 0.053 秒，发生 Full GC（FGC 表示 Full GC）1次，Full GC 总耗时（FGCT 表示 Full GC Time）0.060 秒，所有 GC 总耗时（GCT 表示 GC Time）0.113 秒。

### 主要选项

|选项|作用|
|---|---|
|-class|监视类装载、卸载、总空间以及类装载所耗费的时间|
|-gc|监视 Java 堆状况、包括 Eden 区、两个 Survivor 区、老年代、永久代（或者 mataspace）的容量、已用空间、GC 时间状况等信息|
|-gccapacity|监视内容与 -gc 基本相同，但输出主要关注 Java 堆哥哥区域使用到的最大、最小空间|
|-gcutil|监视内容与 -gc 基本相同，但输出主要关注一使用空间占总空间的百分比|
|-gccause|与 -gcutil 功能一样，但是会额外输出导致上一次 GC 产生的原因|
|-gcnew|监视新生代 GC 状况|
|-gcnewcapacity|监视内容与 -gcnew 基本相同，输出主要关注使用到的最大空间、最小空间|
|-gcold|监视老年代 GC 状况|
|-gcoldcapacity|监视内容与 -gcold 基本相同，输出主要关注使用到的最大空间、最小空间|
|-compiler|输出 JIT 编译器编译过的方法、耗时等信息|
|-printcompilation|输出已经被 JIT 编译的方法|

***命令相关选项及打印信息可参考 [jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html#BEHHGFAE){:target="_blank"}***


## jinfo
jinfo（Configuration Info for Java）的作用是事实查看和调整虚拟机各项参数。使用 `jps -v` 可以查看虚拟机启动时显示指定的参数列表，但如果想知道未被显示指定的参数的系统默认值，除了查资料，就只能使用 jinfo 的 -flag 选项进行查询了，jinfo 还可以使用 -sysprops 选项把虚拟机进程的 System.getProperties() 的内容打印出来。

### 命令格式

```bash
jinfo [option] pid
```

### 执行样例

```bash
➜  ~ jinfo -flag CMSInitiatingOccupancyFraction 5838
-XX:CMSInitiatingOccupancyFraction=-1
➜  ~ 
```

### 主要选项

|选项|作用|
|---|---|
|-flag &lt;name>|打印虚拟机参数的值|
|-flag [+/-] &lt;name>|启用或者禁用虚拟机参数|
|-flag &lt;name>=&lt;value>|设置虚拟机参数的值|
|-flags|打印虚拟机所有参数值|
|-sysprops|打印 Java 系统属性|
|no option|打印虚拟机参数和 Java 系统属性|


***命令相关选项及打印信息可参考 [jinfo](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html#BCGEBFDD){:target="_blank"}***


## jmap

jmap（Memory Map for Java）命令用于生成堆转储快照（一般称为 heapdump 或 dump 文件）。如果不使用 jmap 命令，想要获取 Java 堆转储快照，还有一些比较“暴力”的手段：
1. 使用 -XX:+HeapDumpOnOutOfMemoryError 参数，可以让虚拟机在 OOM 异常出现之后自动生成 dump 文件。
2. 通过 —XX:HeapDumpOnCtrlBreak 参数可以使用 [Ctrl]+[Break] 键让虚拟机生成 dump 文件
3. 在 Linux 系统下通过 kill -3 命令发送进程退出信号“吓唬”一下虚拟机，也能拿到 dump 文件

jamp 的作用并不仅仅是为了获取 dump 文件，它还可以查询 finalize 执行队列、Java 堆的详细信息，如空间使用率、当前用的是哪种收集器等。

### 命令格式

```bash
jmap [option] vmid
```

### 执行样例

```bash
➜  ~ jmap -dump:format=b,file=test.bin 5838
Dumping heap to /Users/xxx/test.bin ...
Heap dump file created
➜  ~ 
```

### 主要选项

|选项|作用|
|---|---|
|-dump|生成 Java 堆转储快照。格式为：-dump:[live,]format=b,file=&lt;filename>，其中 live 子参数说明是否只 dump 出存活对象|
|-finalizerinfo|显示在 F-Queue 中等待 Finalizer 线程执行 finalize 方法的对象。只在 Linux/Solaris 平台下有效|
|-histo|显示堆中对象统计信息，包括类、实例数量、合计容量|
|-F|当虚拟机对 -dump 选项没有响应时，可以使用这个选项强制生成 dump 快照。只在 Liunx/Solaris 平台下有效|

***命令相关选项及打印信息可参考 [jmap](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html#CEGCECJB){:target="_blank"}***

## jhat

Sun JDK 提供 jhat（JVM Heap Analysis Tool）命令与 jmap 搭配使用，来分析 jmap 生成的堆转储快照。jhat 内置一个微型的 HTTP/HTML 服务器，生成 dump 文件的分析结果，可以再浏览器中查看。实际工作中一般不会直接使用 jhat 命令分析 dump 文件。原因有二：
1. 一般不会再部署应用服务器上直接分析 dump 文件，因为一般线上服务不会直接对方开放，无法通过 ip:port 的方式访问，而且分析工作是一个耗时而且消耗硬件资源的过程，既然都要在其他机器上进行，就没有必要收到命令行工具的限制了。
2. jhat 分析功能相对简陋，JDK 提供的 VisualVM 以及专门用于分析 dump 文件的 Eclipse Memory Analyzer、IBM HeapAnalyzer 等工具，都能实现比 jhat 更加强大更加专业的分析功能。

### 执行样例

```bash
➜  ~ jhat test.bin                         
Reading from test.bin...
Dump file created Sun Sep 30 16:43:55 CST 2018
Snapshot read, resolving...
Resolving 1420442 objects...
Chasing references, expect 284 dots............................................................................................................................................................................................................................................................................................
Eliminating duplicate references............................................................................................................................................................................................................................................................................................
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```
分析结果截图如下：

![jhat 的分析结果](/images/jhat-analysis.png)

## jstack

jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的进程快照（一般称为 threaddump 或者 javacore 文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。线程出现停顿的时候通过 jstack 来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情，活着等待着什么资源。

### 命令格式

```bash
jstack [option] vmid
```
### 执行样例

```bash
➜  ~ jstack -l 5838              
2018-09-30 17:04:14
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.131-b11 mixed mode):

"Attach Listener" #22 daemon prio=9 os_prio=31 tid=0x00007fa2989d5800 nid=0x2533 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"qtp323620444-21" #21 prio=5 os_prio=31 tid=0x00007fa2971c3800 nid=0xa503 waiting on condition [0x0000700010ae9000]
   java.lang.Thread.State: TIMED_WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000079621d5d0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
	at org.eclipse.jetty.util.BlockingArrayQueue.poll(BlockingArrayQueue.java:337)
	at org.eclipse.jetty.util.thread.QueuedThreadPool.idleJobPoll(QueuedThreadPool.java:517)
	at org.eclipse.jetty.util.thread.QueuedThreadPool.access$600(QueuedThreadPool.java:39)
	at org.eclipse.jetty.util.thread.QueuedThreadPool$3.run(QueuedThreadPool.java:563)
	at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
	- None

"qtp323620444-20" #20 prio=5 os_prio=31 tid=0x00007fa2984ac000 nid=0xa703 waiting on condition [0x00007000109e6000]
   java.lang.Thread.State: TIMED_WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000079621d5d0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
	at org.eclipse.jetty.util.BlockingArrayQueue.poll(BlockingArrayQueue.java:337)
	at org.eclipse.jetty.util.thread.QueuedThreadPool.idleJobPoll(QueuedThreadPool.java:517)
	at org.eclipse.jetty.util.thread.QueuedThreadPool.access$600(QueuedThreadPool.java:39)
	at org.eclipse.jetty.util.thread.QueuedThreadPool$3.run(QueuedThreadPool.java:563)
	at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
	- None
```

### 主要选项

|选项|作用|
|---|---|
|-F|当正常输出的请求不被响应时，强制输出线程堆栈|
|-l|除了堆栈外，显示关于锁的附加信息|
|-m|如果调用到本地方法的话，可以显示 C/C++ 的堆栈|