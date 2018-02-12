# Java Memory Mode

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/java_mode_1.jpg)

JVM运行时数据区由图所示。



虚拟机将所管理的内存分为以下几个部分：

- 程序计数器
- 虚拟机栈
- 本地方法区
- 堆
- 方法区



其中方法区和堆是由所有线程共享的，例如使用ThreadPoolExecutor创建多个线程时，堆与方法区都可以被多个线程读取。如图：

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/java_mode_2.jpg)



## 1. 程序计数器

程序计数器是一个**概念模型**，各种虚拟机可能会通过一些更高效的方式去实现。在概念模型中，通过计数器的值来选取需要执行的指令，分支，循环，跳转，异常处理，线程恢复等基础功能。

学过计算机组成原理的人都会知道在CPU的寄存器中有一个PC寄存器，存放下一条指令地址，这里，虚拟机不使用CPU的程序计数器，自己在内存中设立一片区域来模拟CPU的程序计数器。只有一个程序计数器是不够的，当多个线程切换执行时，那就单个程序计数器就没办法了，虚拟机规范中指出，每一条线程都有一个独立的程序计数器。注意，**Java虚拟机中的程序计数器指向正在执行的字节码地址，而不是下一条**。





## 2. Java虚拟机栈

Java虚拟机栈也是线程私有的，虚拟机栈描述的是Java方法执行的内存模型：每个方法执行的时候都会创建一个栈帧（我觉得可以把它看作是一个快照，记录下进入方法前的一些参数，实际上是方法运行时的基础数据结构），用于存放局部变量表，操作数栈，动态链接，方法出口等信息。每一个方法从调用直到执行完成的过程都对应着一个栈帧在虚拟机中的入栈到出栈的过程。我们平时把内存分为堆内存和栈内存，其中的栈内存就指的是虚拟机栈的**局部变量表**部分。局部变量表存放了编译期可以知道的基本数据类型，对象引用，和返回后所指向的字节码的地址。

当递归层次太深时，会引发java.lang.StackOverflowError，这是虚拟机栈抛出的异常。





## 3. 本地方法栈

在HotSpot虚拟机将本地方法栈和虚拟机栈合二为一，它们的区别在于，虚拟机栈为执行Java方法服务，而本地方法栈则为虚拟机使用到的Native方法服务





## 4. Java堆

这个区域是用来存放对象实例的，几乎所有对象实例都会在这里分配内存，虚拟机规范中讲：所有对象的实例以及数组都要在堆上分配。但是随着JIT（Just-in-time） 编译期的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术（关于栈上分配和标量替换请参考[JVM中的逃逸分析](http://my.oschina.net/hosee/blog/638573)）将会导致一些微妙的变化发生，所有的对象都分配在堆上也渐渐变得不是那么“绝对”了。堆是Java垃圾收集器管理的主要区域（GC堆），垃圾收集器实现了对象的自动销毁。Java堆可以细分为：新生代和老年代；再细致一点的有Eden空间，From Survivor空间，To Survivor空间等。Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。可以通过-Xmx和-Xms控制



## 5. 方法区

方法区也叫永久代。在过去（自定义类加载器还不是很常见的时候），类大多是”static”的，很少被卸载或收集，因此被称为“永久的(Permanent)”。同时，由于类class是JVM实现的一部分，并不是由应用创建的，所以又被认为是“非堆(non-heap)”内存。

永久代也是各个线程共享的区域，它用于存储已经被虚拟机加载过的类信息，常量，**静态变量（JDK7中被移到Java堆）**，及时编译期编译后的代码（类方法）等数据。这里要讲一下运行时常量池，它是方法区的一部分，用于存放编译期生成的各种字面量和符号引用（其实就是八大基本类型的包装类型和**String类型数据（JDK7中被移到Java堆）**）。

在JDK1.7中的HotASpot中，已经把原本放在方法区的字符串常量池移出，

- 将interned String移到Java堆中
- 将符号Symbols移到native memory（不受GC管理的内存）



从JDK7开始永久代的移除工作，贮存在永久代的一部分数据已经转移到了Java Heap或者是Native Heap。但永久代仍然存在于JDK7，并没有完全的移除：**符号引用(Symbols)转移到了native heap;字面量(interned strings)转移到了java heap;类的静态变量(class statics)转移到了java heap**。



例子：

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<String>();
    int i = 0;
    while (true)
    {
      list.add(String.valueOf(i++).intern());
    }
}
```



在JDK1.6及以前，我们可以通过-XX:PermSize和-XX:MaxPermSize限制方法区大小，从而间接限制其中常量池的容量。

运行结果：

```java
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
```

可以看到是永久代即方法区抛出的异常，说明JDK1.6及以前字符串常量池在方法区呢。

在JDK1.7中，不会出现上述异常。

当设置JVM参数为-Xmx10M -Xms10M时

抛出异常：

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

可以看到是Java堆发出的异常。



同样：

```java
public static void main(String[] args)
{
	String str1 = new StringBuffer("计算机").append("软件").toString();
	System.out.println(str1.intern() == str1);
}
```

上述代码在JDK1.6中输出的是

```java
false
```



因为在JDK1.6中，intern()方法会把首次遇到的字符串实例复制到永久代中，返回的也是永久代中这个字符串实例的引用。而new String是在Java堆中创建对象，固然两者是不同的。

而在JDK1.7中输出的是

```java
true
```



永久代在JDK1.8中进行了巨大改变，JDK1.8中移出了永久代，取而代之的是 Metaspace。

> 在JDK8之前的HotSpot JVM，存放这些”永久的”的区域叫做“永久代(permanent generation)”。永久代是一片连续的堆空间，在JVM启动之前通过在命令行设置参数-XX:MaxPermSize来设定永久代最大可分配的内存空间，默认大小是64M（64位JVM由于指针膨胀，默认是85M）。
>
> **永久代的垃圾收集是和老年代(old generation)捆绑在一起的**，因此无论谁满了，都会触发永久代和老年代的垃圾收集。不过，一个明显的问题是，当JVM加载的类信息容量超过了参数-XX：MaxPermSize设定的值时，应用将会报OOM的错误(32位的JVM默认MaxPermSize是64M，而JDK8里的Metaspace，也可以通过参数-XX:MetaspaceSize 和-XX:MaxMetaspaceSize设定大小，但如果不指定MaxMetaspaceSize的话，**Metaspace的大小仅受限于native memory的剩余大小**。也就是说永久代的最大空间一定得有个指定值，而如果MaxPermSize指定不当，就会OOM)。



随着JDK8的到来，JVM不再有PermGen。但类的元数据信息（metadata）还在，只不过不再是存储在连续的堆空间上，而是移动到叫做“Metaspace”的本地内存（Native memory）中。



由于类的元数据可以在本地内存(native memory)之外分配,所以其最大可利用空间是整个系统内存的可用空间。这样，你将不再会遇到OOM错误，溢出的内存会涌入到交换空间（swap）。最终用户可以为类元数据指定最大可利用的本地内存空间，JVM也可以增加本地内存空间来满足类元数据信息的存储。



## 6. 直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。但是这部分内存频繁地使用，所以这里也提一下。

在JDK1.4版本中加入了NIO类，引入了基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数直接分配堆外内存，也就是说通过这种方式，不会在运行时数据区域分配内存，这样就避免了在运行时数据区域来回复制数据，直接调用外部内存。

显然，本机直接内存的分配不会受到Java堆大小的限制，但是，既然是内存，肯定还是会受到本机总内存大小以及处理器寻址空间的限制。




