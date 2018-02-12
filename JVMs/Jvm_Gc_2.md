# JVM GC 2

之前一篇[JVM_GC_1文章](https://github.com/WhiteSmithFloyd/Java_Basic/blob/master/JVMs/Jvm_Gc_1.md)已经将GC的机制以及GC的算法讲了一下。

而这篇Blog主要是讨论这些GC的算法在JVM中的不同应用。

## 1. 串行收集器

串行收集器是最古老，最稳定以及效率高的收集器
可能会产生较长的停顿，只使用一个线程去回收
-XX:+UseSerialGC

- 新生代、老年代使用串行回收
- 新生代复制算法
- 老年代标记-压缩

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_2_1.jpg)

串行收集器的日志输出：

```tex
0.844: [GC 0.844: [DefNew: 17472K->2176K(19648K), 0.0188339 secs] 17472K->2375K(63360K), 0.0189186 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]

8.259: [Full GC 8.259: [Tenured: 43711K->40302K(43712K), 0.2960477 secs] 63350K->40302K(63360K), [Perm : 17836K->17836K(32768K)], 0.2961554 secs] [Times: user=0.28 sys=0.02, real=0.30 secs]
```



## 2. 并行收集器

### 2.1 ParNew

-XX:+UseParNewGC（new代表新生代，所以适用于新生代）

- 新生代并行
- 老年代串行

Serial收集器新生代的并行版本
在新生代回收时使用复制算法
多线程，需要多核支持
-XX:ParallelGCThreads 限制线程数量

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_2_2.jpg)

并行收集器的日志输出：

```tex
0.834: [GC 0.834: [ParNew: 13184K->1600K(14784K), 0.0092203 secs] 13184K->1921K(63936K), 0.0093401 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

```





### 2.2 Parallel收集器

 类似ParNew 

新生代复制算法 

老年代标记-压缩 

更加关注吞吐量

 -XX:+UseParallelGC  

- 使用Parallel收集器+ 老年代串行

 -XX:+UseParallelOldGC 

- 使用Parallel收集器+ 老年代并行

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_2_3.jpg)

Parallel收集器的日志输出：

```tex
1.500: [Full GC [PSYoungGen: 2682K->0K(19136K)] [ParOldGen: 28035K->30437K(43712K)] 30717K->30437K(62848K) [PSPermGen: 10943K->10928K(32768K)], 0.2902791 secs] [Times: user=1.44 sys=0.03, real=0.30 secs]

```



### 2.3 其他GC参数

**-XX:MaxGCPauseMills**

- 最大停顿时间，单位毫秒
- GC尽力保证回收时间不超过设定值

**-XX:GCTimeRatio** 

- 0-100的取值范围
- 垃圾收集时间占总时间的比
- 默认99，即最大允许1%时间做GC

这两个参数是矛盾的。因为停顿时间和吞吐量不可能同时调优



## 3. CMS收集器

- Concurrent Mark Sweep **并发**标记清除（应用程序线程和GC线程交替执行）
- **使用标记-清除算法**
- 并发阶段会降低吞吐量（停顿时间减少，吞吐量降低）
- **老年代收集器**（新生代使用ParNew）
- -XX:+UseConcMarkSweepGC



CMS运行过程比较复杂，着重实现了标记的过程，可分为

1. **初始标记（会产生全局停顿）**

- 根可以直接关联到的对象
- 速度快

2. **并发标记（和用户线程一起）** 

- 主要标记过程，标记全部对象

3. **重新标记 （会产生全局停顿）**

 由于并发标记时，用户线程依然运行，因此在正式清理前，再做修正

4. **并发清除（和用户线程一起）** 

- 基于标记结果，直接清理对象

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_2_4.jpg)

这里就能很明显的看出，为什么CMS要使用标记清除而不是标记压缩，如果使用标记压缩，需要多对象的内存位置进行改变，这样程序就很难继续执行。但是标记清除会产生大量内存碎片，不利于内存分配。 

CMS收集器的日志输出：

```tex
1.662: [GC [1 CMS-initial-mark: 28122K(49152K)] 29959K(63936K), 0.0046877 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
1.666: [CMS-concurrent-mark-start]
1.699: [CMS-concurrent-mark: 0.033/0.033 secs] [Times: user=0.25 sys=0.00, real=0.03 secs] 
1.699: [CMS-concurrent-preclean-start]
1.700: [CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
1.700: [GC[YG occupancy: 1837 K (14784 K)]1.700: [Rescan (parallel) , 0.0009330 secs]1.701: [weak refs processing, 0.0000180 secs] [1 CMS-remark: 28122K(49152K)] 29959K(63936K), 0.0010248 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
1.702: [CMS-concurrent-sweep-start]
1.739: [CMS-concurrent-sweep: 0.035/0.037 secs] [Times: user=0.11 sys=0.02, real=0.05 secs] 
1.739: [CMS-concurrent-reset-start]
1.741: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```



**CMS收集器特点：**

尽可能降低停顿
会影响系统整体吞吐量和性能

>  比如，在用户线程运行过程中，分一半CPU去做GC，系统性能在GC阶段，反应速度就下降一半

清理不彻底 

> 因为在清理阶段，用户线程还在运行，会产生新的垃圾，无法清理

因为和用户线程一起运行，不能在空间快满时再清理（因为也许在并发GC的期间，用户线程又申请了大量内存，导致内存不够） 

> - -XX:CMSInitiatingOccupancyFraction设置触发GC的阈值
> - 如果不幸内存预留空间不够，就会引起concurrent mode failure



```tex
33.348: [Full GC 33.348: [CMS33.357: [CMS-concurrent-sweep: 0.035/0.036 secs] [Times: user=0.11 sys=0.03, real=0.03 secs] 
 (concurrent mode failure): 47066K->39901K(49152K), 0.3896802 secs] 60771K->39901K(63936K), [CMS Perm : 22529K->22529K(32768K)], 0.3897989 secs] [Times: user=0.39 sys=0.00, real=0.39 secs]

```

一旦 concurrent mode failure产生，将使用串行收集器作为后备。



**CMS也提供了整理碎片的参数：**

-XX:+ UseCMSCompactAtFullCollection Full GC后，进行一次整理

> 整理过程是独占的，会引起停顿时间变长

-XX:+CMSFullGCsBeforeCompaction  

> 设置进行几次Full GC后，进行一次碎片整理

-XX:ParallelCMSThreads 

> 设定CMS的线程数量（一般情况约等于可用CPU数量）

CMS的提出是想改善GC的停顿时间，在GC过程中的确做到了减少GC时间，但是同样导致产生大量内存碎片，又需要消耗大量时间去整理碎片，从本质上并没有改善时间。



## 4. G1收集器

G1是目前技术发展的最前沿成果之一，HotSpot开发团队赋予它的使命是未来可以替换掉JDK1.5中发布的CMS收集器。

与CMS收集器相比G1收集器有以下特点：

1. 空间整合，G1收集器采用标记整理算法，不会产生内存空间碎片。分配大对象时不会因为无法找到连续空间而提前触发下一次GC。
2. 可预测停顿，这是G1的另一大优势，降低停顿时间是G1和CMS的共同关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为N毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒，这几乎已经是实时Java（RTSJ）的垃圾收集器的特征了。



上面提到的垃圾收集器，收集的范围都是整个新生代或者老年代，而G1不再是这样。使用G1收集器时，Java堆的内存布局与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔阂了，它们都是一部分（可以不连续）Region的集合。

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_2_5.jpg)

G1的新生代收集跟ParNew类似，当新生代占用达到一定比例的时候，开始出发收集。

和CMS类似，G1收集器收集老年代对象会有短暂停顿。

步骤：

1. 标记阶段，首先初始标记(Initial-Mark),这个阶段是停顿的(Stop the World Event)，并且会触发一次普通Mintor GC。对应GC log:GC pause (young) (inital-mark)
2. Root Region Scanning，程序运行过程中会回收survivor区(存活到老年代)，这一过程必须在young GC之前完成。
3. Concurrent Marking，在整个堆中进行并发标记(和应用程序并发执行)，此过程可能被young GC中断。在并发标记阶段，若发现区域对象中的所有对象都是垃圾，那个这个区域会被立即回收(图中打X)。同时，并发标记过程中，会计算每个区域的对象活性(区域中存活对象的比例)。

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_2_6.jpg)

4. Remark, 再标记，会有短暂停顿(STW)。再标记阶段是用来收集 并发标记阶段 产生新的垃圾(并发阶段和应用程序一同运行)；G1中采用了比CMS更快的初始快照算法:snapshot-at-the-beginning (SATB)。

5. Copy/Clean up，多线程清除失活对象，会有STW。G1将回收区域的存活对象拷贝到新区域，清除Remember Sets，并发清空回收区域并把它返回到空闲区域链表中。

   ![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_2_7.jpg)

6. 复制/清除过程后。回收区域的活性对象已经被集中回收到深蓝色和深绿色区域。


![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_2_8.jpg)



## 5. 安全点

GC的停顿主要来源于可达性分析上，程序执行时并非在所有地方都能停顿下来开始GC，只有在到达安全点时才能暂停。

安全点的选定基本上是以程序“是否具有让程序长时间执行的特征”为标准进行选定的——因为每条指令执行的时间都非常短暂，程序不太可能因为指令流长度太长这个原因而过长时间运行，“长时间执行”的最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等，所以具有这些功能的指令才会产生安全点。

接下来的问题就在于，如何让程序在需要GC时都跑到安全点上停顿下来，大多数JVM的实现都是采用主动式中断的思想。

主动式中断的思想是当GC需要中断线程的时候，不直接对线程操作，仅仅简单地设置一个标志，各个线程执行时主动去轮询这个标志，发现中断标志为真时就自己中断挂起，轮询标志的地方和安全点是重合的，另外再加上创建对象需要分配内存的地方。



## 其他：

只要记住流行的组合就这几种情况
Serial
ParNew + CMS
ParallelYoung + ParallelOld
G1GC




