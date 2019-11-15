# JVM 性能调优

> JVM 本身给我们提供了很多强大而有效的监控进程、分析定位瓶颈的工具，比如 JConsole、JMap、JStack、JStat 等等。
>
> 使用这些命令，再结合 Linux 自身提供的一些强大的进程、线程命令，能够快速定位系统瓶颈

本文背景：

- Linux：RedHat 6.1
- Weblogic：11g
- JRokit：R28.2.4
- JDK：1.6.0_45



### LoadRunner 压测结果

50 个用户并发压十几个小时，TRT 在 20 左右，TPS 在 2.5 左右



### 分析数据库的压力

分析数据库性能，使用`Spotlight`查看数据库服务器的压力，判断性能瓶颈是否在数据库端

> Spotlight下载地址：
>
> <http://spotlight-on-unix.software.informer.com/download/#downloading>



### Weblogic 所在的 Linux 服务器的压力

使用`Spotlight` 查看服务器在并发情况下的压力情况

> 虽然 CPU 利用率始终在 200% 左右徘徊，但这对于 16 核的系统来讲也算是正常



###  JStack 报告分析

`JStack` 能够看到当前 Java 进程中每个线程的**当前状态**、**调用栈**、**锁住**或**等待去锁定的资源**，而且很强悍的是它还能直接报告**是否有线程死锁**

#### JStack 拉取的文件信息基本分为以下几个部分：

- 该拉取快照的服务器时间
- JVM 版本
- 以线程 ID (即 tid) 升序依次列出当前进程中每个线程的调用栈
- 死锁 (如果有的话)
- 阻塞锁链
- 打开的锁链
- 监视器解锁情况跟踪

**每个线程在等待什么资源，这个资源目前在被哪个线程 hold，尽在眼前**。JStack 最好在压测时多次获取，找到的普遍存在的现象即为线程瓶颈所在。



#### TLA 空间的调整

**TLA** 是 thread local area 的缩写，是每个线程私有的空间，所以在多线程环境下 TLA 带来的性能提升是显而易见的

```java
// 多次拉取 JStack，发现很多线程处于这个状态：
    at jrockit/vm/Allocator.getNewTla(JJ)V(Native Method)
    at jrockit/vm/Allocator.allocObjectOrArray(Allocator.java:354)[optimized]
    at java/util/HashMap.resize(HashMap.java:564)[inlined]
    at java/util/LinkedHashMap.addEntry(LinkedHashMap.java:414)[optimized]
    at java/util/HashMap.put(HashMap.java:477)[optimized]
// 由此怀疑出现上述堆栈的原因可能是 TLA 空间不足引起。
/**
 * TLA 默认最小大小 2 KB，默认首选大小 16 KB - 256 KB (取决于新生代分区大小)。
 	这里我们调整 TLA 空间大小为最小 32 KB，首选 1024 KB，JVM 启动参数中加入：
	-XXtlaSize:min=32k,preferred=1024k
 */
```

如果大部分线程的需要分配的对象都较大，可以考虑提高 TLA 空间，因为这样更大的对象可以在 TLA 中进行分配，这样就不用担心和其它线程的同步问题了。

但这个也不可以调的太大，否则也会带来一些问题，比如会带来更多内存碎片、更加频繁的垃圾搜集。



### JStat 结合 GC 日志报告分析

> TLA 频繁申请是降下来了，但 TRT 仍旧是 20，TPS 依旧 2.5。别灰心，改一个地方就立竿见影，胜利似乎来得太快了点

现在怀疑是服务堆内存太小，查看一下果然。服务器物理内存 32 GB，Weblogic 进程只分到了 6 GB。怎么查看？至少有四种办法：

+ ##### ps 命令

  ```shell
  ps -ef | grep java
  defonds     29874 29819  2 Sep03 ?        09:03:17 /opt/jrockit-jdk1.6.0_33/bin/java -jrockit -Xms6000m -Xmx6000m -Dweblogic.Name=AdminServer -Djava.security.policy=
  ```

+ ##### JStat 报告


  ```shell
  jstat -J-Djstat.showUnsupported=true -snap 10495 > ~/10495jstat.txt

  ```

  注意这个信息是一个快照，这是跟 GC 日志报告不同的地方。

  ```txt
  jrockit.gc.latest.heapSize=10737418240
  jrockit.gc.latest.nurserySize=23100384
  ```

  上述是当前已用碓大小和新生代分区大小。多拉几次即可估算出各自分配的大小

+ #####GC日志

  ```shell
  # 在Java进程启动参数里面加入
  -Xverbose:memory -Xverboselog:verboseText.txt
  ```

  `verboseText.txt` 文件中

  ```shell
  # maximal heap size 即为该进程分配到的最大堆大小
  [INFO ][memory ] Heap size: 10485760KB, maximal heap size: 10485760KB, nursery size: 5242880KB.
  # 下面还有进程启动以后较为详细的每次 GC 的信息：
  [INFO ][memory ] [YC#2547] 340.828-340.845: YC 10444109KB->10417908KB (10485760KB), 0.018 s, sum of pauses 17.294 ms, longest pause 17.294 ms.
  [INFO ][memory ] [YC#2548] 340.852-340.871: YC 10450332KB->10434521KB (10485760KB), 0.019 s, sum of pauses 18.779 ms, longest pause 18.779 ms.
  [INFO ][memory ] [YC#2549] 340.878-340.895: YC 10476739KB->10485760KB (10485760KB), 0.017 s, sum of pauses 16.520 ms, longest pause 16.520 ms.
  [INFO ][memory ] [OC#614] 340.895-341.126: OC 10485760KB->10413562KB (10485760KB), 0.231 s, sum of pauses 206.458 ms, longest pause 206.458 ms.

  # 第一行表示该进程启动后的第 340.828 秒 - 340.845 秒期间进行了一次 young gc，该次 GC 持续了 17.294 ms，将整个已用掉的堆内存由 10444109 KB 降低到 10417908 KB。

  # 第三行同样是一次 young gc，但该次 GC 后已用堆内存反而上升到了 10485760 KB，也就是达到最大堆内存，于是该次 young gc 结束的同时触发 full gc。

  # 第四行是一次 old gc (即 full gc)，将已用堆内存由 10485760 KB 降到了 10413562 KB，耗时 206.458 ms。
  # 这些日志同样能够指出当前压力下的 GC 的频率，为本次性能瓶颈 - GC 过于频繁提供有力的佐证。
  ```

  ​

#### 调整内存

根据 `ps`命令我们得出当前服务器分配堆内存太小的结论

根据 `GC` 日志报告和 `JStat` 报告可以得出新生代分区太小的结论。

于是我们调整它们的大小，结合 4.1 TLA 调整的结论，JVM 启动参数增加以下：

`-Xms10240m -Xmx10240m -Xns:1024m -XXtlaSize:min=32k, preferred=1024k`

再次压测，`TRT` 降到了 2.5，`TPS` 上升到 20。


### 性能瓶颈的定位

#### 性能线程的定位

+ 使用`TOP` 拿到CPU最高的进程ID

+ 性能线程的获取

  ``` shell
  ps p 10495 -L -o pcpu,pid,tid,time,tname,cmd > ~/10495ps.txt
  # 多次拉这个快照，我们找到了 tid 为 10499、10500、10501、10502 等线程一直在占用着很高的 CPU
  ```

+ `Jstack` 抓取快照看看都是什么线程慢 

  `jstack -l 10495 > ~/10495jstack.txt`

  ``` shell
  "(GC Worker Thread 1)" id=? idx=0x10 tid=10499 prio=5 alive, daemon
      at pthread_cond_wait@@GLIBC_2.3.2+202(:0)@0x3708c0b44a
      at eventTimedWaitNoTransitionImpl+71(event.c:90)@0x7fac47be8528
      at eventTimedWaitNoTransition+66(event.c:72)@0x7fac47be8593
      at mmGCWorkerThread+137(gcthreads.c:809)@0x7fac47c0774a
      at thread_stub+170(lifecycle.c:808)@0x7fac47cc15bb
      at start_thread+208(:0)@0x3708c077e1
      Locked ownable synchronizers:
          - None

  "(GC Worker Thread 2)" id=? idx=0x14 tid=10500 prio=5 alive, daemon
      at pthread_cond_wait@@GLIBC_2.3.2+202(:0)@0x3708c0b44a
      at eventTimedWaitNoTransitionImpl+71(event.c:90)@0x7fac47be8528
      at eventTimedWaitNoTransition+66(event.c:72)@0x7fac47be8593
      at mmGCWorkerThread+137(gcthreads.c:809)@0x7fac47c0774a
      at thread_stub+170(lifecycle.c:808)@0x7fac47cc15bb
      at start_thread+208(:0)@0x3708c077e1
      Locked ownable synchronizers:
          - None

  "(GC Worker Thread 3)" id=? idx=0x18 tid=10501 prio=5 alive, daemon
      at pthread_cond_wait@@GLIBC_2.3.2+202(:0)@0x3708c0b44a
      at eventTimedWaitNoTransitionImpl+71(event.c:90)@0x7fac47be8528
      at eventTimedWaitNoTransition+66(event.c:72)@0x7fac47be8593
      at mmGCWorkerThread+137(gcthreads.c:809)@0x7fac47c0774a
      at thread_stub+170(lifecycle.c:808)@0x7fac47cc15bb
      at start_thread+208(:0)@0x3708c077e1
      Locked ownable synchronizers:
          - None

  "(GC Worker Thread 4)" id=? idx=0x1c tid=10502 prio=5 alive, daemon
      at pthread_cond_wait@@GLIBC_2.3.2+202(:0)@0x3708c0b44a
      at eventTimedWaitNoTransitionImpl+71(event.c:90)@0x7fac47be8528
      at eventTimedWaitNoTransition+66(event.c:72)@0x7fac47be8593
      at mmGCWorkerThread+137(gcthreads.c:809)@0x7fac47c0774a
      at thread_stub+170(lifecycle.c:808)@0x7fac47cc15bb
      at start_thread+208(:0)@0x3708c077e1
      Locked ownable synchronizers:
          - None
  ```




#### 找到性能瓶颈

事情到了这里，已经不难得出当前系统瓶颈就是频繁 GC。
为何会如此频繁 GC 呢？继续看 JStack，发现这两个互相矛盾的现象：
一方面， **GC Worker 线程在拼命 GC，但是 GC 前后效果不明显，已用堆内存始终降不下来**；
另一方面，**大量 ExecuteThread 业务处理线程处于` alloc_enqueue_allocation_and_wait_for_gc` 的 `native_blocked` 阻塞状态**。

此外，停止压测以后，查看已用堆内存大小，也就几百兆，不到分配堆内存的 1/10。
这说明了什么呢？这说明了我们应用里没有内存泄漏、静态对象不是太多、有大量的业务线程在频繁创建一些生命周期很长的临时对象。


