## 1、 jmap命令简介

jmap（Java Virtual Machine Memory Map）是JDK提供的一个可以生成Java虚拟机的堆转储快照dump文件的命令行工具。除此以外，jmap命令还可以查看finalize执行队列、Java堆和方法区的详细信息，比如空间使用率、当前使用的什么垃圾回收器、分代情况等等。 配合jstack命令使用，通过分析线程堆栈，和内存堆存储的信息，基本上可以分析90%以上线上遇到的问题。

**基本语法**：

> jmap [options] pid

```bash
[app@dzopdev-010010195187 ~]$ jmap -h
Usage:
    jmap [option] <pid>
        (to connect to running process)
    jmap [option] <executable <core>
        (to connect to a core file)
    jmap [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    <none>               to print same info as Solaris pmap
    -heap                to print java heap summary
    -histo[:live]        to print histogram of java object heap; if the "live"
                         suboption is specified, only count live objects
    -clstats             to print class loader statistics
    -finalizerinfo       to print information on objects awaiting finalization
    -dump:<dump-options> to dump java heap in hprof binary format
                         dump-options:
                           live         dump only live objects; if not specified,
                                        all objects in the heap are dumped.
                           format=b     binary format
                           file=<file>  dump heap to <file>
                         Example: jmap -dump:live,format=b,file=heap.bin <pid>
    -F                   force. Use with -dump:<dump-options> <pid> or -histo
                         to force a heap dump or histogram when <pid> does not
                         respond. The "live" suboption is not supported
                         in this mode.
    -h | -help           to print this help message
    -J<flag>             to pass <flag> directly to the runtime system

复制代码
```

## 2. options

### 2.1、none

to print same info as Solaris pmap 不带选项参数，jmap打印共享对象映，打印目标虚拟机中加载的每个共享对象的起始地址、映射大小以及共享对象文件的路径全称。这与Solaris的pmap工具比较相似。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52c66d3736334466b478d7f0761fc41d~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

### 2.2、-heap

打印堆的摘要信息

> 1. 垃圾回收器, Parallel GC
> 2. Heap Configuration 堆配置信息
> 3. Heap Usage 堆内存空间使用信息，包括分代情况，每个代的总容量、已使用内存、可使用内存。如果某一代被继续细分(例如，年轻代，老年代， eden ， from， to 等)，则包含细分的空间的内存使用信息。

```bash
[app@dzopdev-010010195187 ~]$ jmap -heap 3994
Attaching to process ID 3994, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.191-b12

using thread-local object allocation.
Parallel GC with 13 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 1073741824 (1024.0MB)
   NewSize                  = 357564416 (341.0MB)
   MaxNewSize               = 357564416 (341.0MB)
   OldSize                  = 716177408 (683.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 318242816 (303.5MB)
   used     = 73695144 (70.28116607666016MB)
   free     = 244547672 (233.21883392333984MB)
   23.156891623281766% used
From Space:
   capacity = 19398656 (18.5MB)
   used     = 11316704 (10.792449951171875MB)
   free     = 8081952 (7.707550048828125MB)
   58.33756730363176% used
To Space:
   capacity = 18874368 (18.0MB)
   used     = 0 (0.0MB)
   free     = 18874368 (18.0MB)
   0.0% used
PS Old Generation
   capacity = 716177408 (683.0MB)
   used     = 52877440 (50.4278564453125MB)
   free     = 663299968 (632.5721435546875MB)
   7.383287912930088% used

33738 interned Strings occupying 3267696 bytes.
[app@dzopdev-010010195187 ~]$
复制代码
```

### 2.3、-histo[:live]

显示Java堆中对象的统计信息，包括：对象数量、占用内存大小(单位：字节)和类的完全限定名。 这边我直接看的开发环境的进程比较多，重定向到日志的log文件里面了。

```bash
[app@dzopdev-010010195187 ~]$ jmap -histo 6770 >jmap.6770.log
[app@dzopdev-010010195187 ~]$ vi jmap.6770.log


 num     #instances         #bytes  class name
----------------------------------------------
   1:         51151      250484560  [I
   2:        181787       26623120  [C
   3:         33083       15735520  [B
   4:        146598        3518352  java.lang.String
   5:         86394        3455760  ch.qos.logback.core.status.InfoStatus
   6:         36608        1933944  [Ljava.lang.Object;
   7:         57458        1838656  java.util.concurrent.ConcurrentHashMap$Node
   8:         20534        1806992  java.lang.reflect.Method
   9:         13462        1485000  java.lang.Class
  10:         31080        1243200  java.util.LinkedHashMap$Entry
  11:         14651        1133808  [Ljava.util.HashMap$Node;
  12:         57482         919712  java.lang.Object
  13:         25192         806144  java.util.HashMap$Node
  14:         13312         745472  java.util.LinkedHashMap
  15:         28529         684696  java.util.ArrayList
  16:          2654         683760  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  17:         24930         550632  [Ljava.lang.Class;
  18:         15830         538880  [Ljava.lang.String;
  19:          4500         504000  java.net.SocksSocketImpl
  20:          6606         475632  java.lang.reflect.Field
  21:            79         324848  [Ljava.nio.ByteBuffer;
  22:         11726         281424  java.util.concurrent.ConcurrentLinkedQueue$Node
  23:          5421         260208  java.util.HashMap
  24:          3204         256320  java.lang.reflect.Constructor
  25:          2020         242400  org.springframework.boot.loader.jar.JarEntry
  26:          7413         237216  java.util.concurrent.locks.AbstractQueuedSynchronizer$Node
  27:          7060         225920  java.util.ArrayList$Itr
  28:          4491         215568  java.net.SocketInputStream
  29:          4491         215568  java.net.SocketOutputStream
  30:          4051         194448  java.nio.HeapCharBuffer
  31:          6071         194272  java.net.InetAddress$InetAddressHolder
  32:          8005         192120  java.lang.StringBuilder
  33:          4794         191760  java.lang.ref.SoftReference
  34:          2849         182336  java.util.concurrent.ConcurrentHashMap
"jmap.6770.log" 5896L, 559623C 
复制代码
```

如果指定了`live`参数，则只计算活动的对象.

```bash
[app@dzopdev-010010195187 ~]$ jmap -histo:live 6770 >jmap.live.6770.log
[app@dzopdev-010010195187 ~]$ vi jmap.live.6770.log


 num     #instances         #bytes  class name
----------------------------------------------
   1:         79166        9245160  [C
   2:         14760        7002048  [I
   3:          8982        4487632  [B
   4:         78290        1878960  java.lang.String
   5:         49088        1570816  java.util.concurrent.ConcurrentHashMap$Node
   6:         13283        1466384  java.lang.Class
   7:         23524         940960  java.util.LinkedHashMap$Entry
   8:         16569         909536  [Ljava.lang.Object;
   9:          9495         751088  [Ljava.util.HashMap$Node;
  10:         39881         638096  java.lang.Object
  11:          6384         561792  java.lang.reflect.Method
  12:          9706         543536  java.util.LinkedHashMap
  13:           299         473472  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  14:         14304         457728  java.util.HashMap$Node
  15:          1968         220416  java.net.SocksSocketImpl
  16:          9000         216000  java.util.ArrayList
  17:          1593         191160  org.springframework.boot.loader.jar.JarEntry
  18:            37         152144  [Ljava.nio.ByteBuffer;
  19:          3671         144552  [Ljava.lang.String;
  20:          6449         138912  [Ljava.lang.Class;
  21:          3413         136520  java.lang.ref.SoftReference
  22:          2649         127152  java.util.HashMap
  23:          2453         117744  org.apache.tomcat.util.buf.ByteChunk
  24:          2925         117000  java.lang.ref.Finalizer
  25:          1618         116496  org.springframework.core.annotation.AnnotationAttributes
  26:          2268         108864  org.apache.tomcat.util.buf.CharChunk
  27:          2207         105936  org.apache.tomcat.util.buf.MessageBytes
  28:          3270         104640  java.util.LinkedList
  29:          4106          98544  java.util.LinkedList$Node
  30:          1959          94032  java.net.SocketInputStream
  31:          1959          94032  java.net.SocketOutputStream
  32:          1100          88000  java.lang.reflect.Constructor
  33:          2071          82840  java.util.TreeMap$Entry
  34:          1431          80136  java.lang.invoke.MemberName
"jmap.liev.6770.log" 5489L, 525768C
复制代码
```

### 2.4、 -clstats

显示Java堆中元空间的类加载器的统计信息，to print class loader statistics。

```bash
[app@dzopdev-010010195187 ~]$ jmap -clstats 6770 >jmap.clstats.6770.log
finding class loader instances ..done.
computing per loader stat ..done.
please wait.. computing liveness.liveness analysis may be inaccurate ...
[app@dzopdev-010010195187 ~]$ vim jmap.clstats.6770.log

Attaching to process ID 6770, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.191-b12
class_loader    classes bytes   parent_loader   alive?  type

<bootstrap>     3059    5221648   null          live    <internal>
0x00000000e1908890      1       1474    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1a8aaa8      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e009a608      1       1471    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1524b50      1       1476    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1b217b0      1       894     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e0def4d8      1       1474    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1a8b3a0      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e20eca18      1       1472      null          dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e05fb748      1       1473    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e0705268      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1c1e3d8      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1524448      1       1473    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1528d48      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e19089b0      1       1474    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1a8b788      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e2301310      1       1474    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e152ae70      1       1495      null          dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1a8b080      1       1482    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e22ada00      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e22b2400      1       1475      null          dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e0dda8f0      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e0def0f0      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e0796948      1       880     0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e04a6c78      1       1473    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e05fb268      1       1473    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1524768      1       1473    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1526668      1       1474    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e1b21a88      1       1471    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e0dda4e0      1       1474    0x00000000e0005148      dead    sun/reflect/DelegatingClassLoader@0x000000010000a028
0x00000000e000a840      48      81649   0x00000000e000a8a0      dead    sun/misc/Launcher$AppClassLoader@0x000000010000f8d8
"jmap.clstats.6770.log" 144L, 14061C 
复制代码
```

### 2.5、-finalizerinfo

to print information on objects awaiting finalization

```bash
[app@dzopdev-010010195187 ~]$ jmap -finalizerinfo 6770
Attaching to process ID 6770, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.191-b12
Number of objects pending for finalization: 0
[app@dzopdev-010010195187 ~]$ 
复制代码
```

### 2.6、-dump:

Example: jmap -dump:live,format=b,file=heap.bin  生成Java虚拟机的堆转储快照dump文件。一般上线出现问题了，通过日志无法分析问题的原因，会联系运维拉去堆快照dump文件，方便问题的定位，或者jvm参数指定内存溢出的转储dump文件。

```bash
[app@dzopdev-010010195187 ~]$ jmap -dump:live,format=b,file=/home/app/heap.6770.bin 6770
Dumping heap to /home/app/heap.6770.bin ...
Heap dump file created
[app@dzopdev-010010195187 ~]$ 
复制代码
```

dump文件如何分析，当然我们主要是借助一些分析工具去做的分析，比如jprofile。

> **注意：** 
>  一般发生致命问题后，Java进程有时可以继续运行，但有时会挂掉。为了能够保留Java应用发生致命错误前的运行状态，JVM在死掉前产生两个文件，分别为JavaCore及HeapDump文件，便于后面问题分析和定位。
>  JavaCore和Heapdump区别后面会详细介绍一下，JavaCode主要是分析CPU的，比如程序在某个点卡死的现象。JavaCode文件主要保存的是Java应用各线程在某一时刻的运行的位置。

### 2.7、-F

强制模式。如果指定的pid没有响应，可以配合`-dump`或`-histo`一起使用。此模式下，不支持live参数。



作者：xxbb
链接：https://juejin.cn/post/7103532982283026439
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。