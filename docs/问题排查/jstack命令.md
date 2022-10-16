## jstack命令介绍

`jstack`（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合.生成线程快照的主要目的是**定位线程出现长时间停顿的原因**，如**线程间死锁、死循环、请求外部资源导致的长时间等待**等。

**基本语法格式**

jstack   pid我们可以通过jps命令或者ps -ef|grep java 等找到我们应用进程。

```bash
jstack [ option ] pid
jstack [ option ] executable core
jstack [ option ] [server-id@]remote-hostname-or-IP
复制代码
```

> **OPTIONS**
>
> - **-F** Force a stack dump when 'jstack [-l] pid' does not respond. 当 jstack [-l] pid 没有响应时，强制打印一个堆栈转储。注意这里区分大小写的。
> - **-l** 打印关于锁的其他信息，比如拥有的java.util.concurrent ownable同步器的列表。
> - **-m** prints mixed mode (both Java and native C/C++ frames) stack trace. 打印包含Java和本机C/ C++帧的混合模式堆栈跟踪。
> - **-h** 打印帮助信息
> - **-help** 打印帮助信息

> **注意：** 
>  1、 不通虚拟机的dump线程的创建方和格式是不一样的。不同jvm版本的dump信息也是有差异的。
>  2、 在实际运行中，往往一次 dump的信息，还不足以确认问题。建议产生三次 dump信息，如果每次dump都指向同一个问题，我们才确定问题的典型性, 当然也需要配合其他的监控信息确定dump的时机，比如top命令或者jstat查看系统gc的情况。

## jstack日志剖析

在分析线程堆栈快照日志我们需要了解线程状态的流传基本知识

### 1、线程状态的状态

> **NEW**,未启动的。不会出现在Dump中。
>  **RUNNABLE,** 在虚拟机内执行的。运行中状态，可能里面还能看到locked字样，表明它获得了某把锁。
>  **BLOCKED**,受阻塞并等待监视器锁。被某个锁(synchronizers)給block住了。
>  **WATING**,无限期等待另一个线程执行特定操作。等待某个condition或monitor发生，一般停留在park(), wait(), sleep(),join() 等语句里。
>  **TIMED_WATING,** 有时限的等待另一个线程的特定操作。和WAITING的区别是wait() 等语句加上了时间限制 wait(timeout)。
>  **TERMINATED**,已退出的。

### 2、Monitor

Monitor是 Java中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者 Class的锁。每一个对象都有，也仅有一个 monitor。下 面这个图，描述了线程和 Monitor之间关系，以 及线程的状态转换图：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d24f125e90d940fbb020c8a899d58f7a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

- **进入区(Entrt Set)** :表示线程通过synchronized要求获取对象的锁。如果对象未被锁住,则迚入拥有者;否则则在进入区等待。一旦对象锁被其他线程释放,立即参与竞争。
- **拥有者(The Owner)** :表示某一线程成功竞争到对象锁。
- **等待区(Wait Set)** :表示线程通过对象的wait方法,释放对象的锁,并在等待区等待被唤醒。

**Monitor锁竞争的主要流程**:
 一个 Monitor在某个时刻，只能被一个线程拥有，该线程就是 “Active Thread”，而其它线程都是 “Waiting Thread”，分别在两个队列 “ Entry Set”和 “Wait Set”里面等候。 在 “Entry Set”中等待的线程状态是 “Waiting for monitor entry”，而在 “Wait Set”中等待的线程状态是 “in Object.wait()”。

“Entry Set”里面的线程。我们称被 synchronized保护起来的代码段为临界区。当一个线程申请进入临界区时，它就进入了 “Entry Set”队列。

### 3、jstack dump文件中线程的状态

> **Deadlock** 死锁, (`重点关注`) , 死锁线程，一般指多个线程调用间，进入相互资源占用，导致一直等待无法释放的情况。
>  **Runnable** 执行中
>  **Waiting on condition** 等待资源, 该状态出现在线程等待某个条件的发生。 (`重点关注`) 
>  **Waiting on monitor entry** 等待获取监视器 , Monitor是 Java中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者 Class的锁。每一个对象都有，也仅有一个 monitor。(`重点关注`) 
>  **Suspended** 暂停状态  
>  **Object.wait()** 或者 **TIMED_WAITING** 
>  **Blocked** 阻塞线程 (`重点关注`), 线程阻塞，是指当前线程执行过程中，所需要的资源长时间等待却一直未能获取到，被容器的线程管理器标识为阻塞状态，可以理解为等待资源超时的线程。 
>  **Parked**



作者：xxbb
链接：https://juejin.cn/post/7103146736830398495
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。