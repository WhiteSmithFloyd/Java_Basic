# 高并发 - Java内存模型和线程安全

网上很多资料在描述Java内存模型的时候，都会介绍有一个主存，然后每个工作线程有自己的工作内存。数据在主存中会有一份，在工作内存中也有一份。工作内存和主存之间会有各种原子操作去进行同步。

下图来源于

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_1.jpg)

但是由于Java版本的不断演变，内存模型也进行了改变。本文只讲述Java内存模型的一些特性，无论是新的内存模型还是旧的内存模型，在明白了这些特性以后，看起来也会更加清晰。



## 1. 原子性

- 原子性是指一个操作是不可中断的。即使是在多个线程一起执行的时候，一个操作一旦开始，就不会被其它线程干扰。

一般认为cpu的指令都是原子操作，但是我们写的代码就不一定是原子操作了。

比如说i++。这个操作不是原子操作，基本分为3个操作，读取i，进行+1，赋值给i。

假设有两个线程，当第一个线程读取i=1时，还没进行+1操作，切换到第二个线程，此时第二个线程也读取的是i=1。随后两个线程进行后续+1操作，再赋值回去以后，i不是3，而是2。显然数据出现了不一致性。

再比如在32位的JVM上面去读取64位的long型数值，也不是一个原子操作。当然32位JVM读取32位整数是一个原子操作。



## 2. 有序性

- 在并发时，程序的执行可能就会出现乱序。

计算机在执行代码时，不一定会按照程序的顺序来执行。

```java
class OrderExample { 
		int a = 0; 
		boolean flag = false; 
		public void writer() 
		{ 
			a = 1; 
			flag = true; 
		} 
		public void reader() 
		{ 
			if (flag) 
			{ 
				int i = a +1;  
			}
		} 
	}
```

比如上述代码，两个方法分别被两个线程调用。按照常理，写线程应该先执行a=1，再执行flag=true。当读线程进行读的时候，i=2；

但是因为a=1和flag=true，并没有逻辑上的关联。所以有可能执行的顺序颠倒，有可能先执行flag=true，再执行a=1。这时当flag=true时，切换到读线程，此时a=1还没有执行，那么读线程将i=1。

当然这个不是绝对的。是有可能会发生乱序，有可能不发生。

那么为什么会发生乱序呢？这个要从cpu指令说起，Java中的代码被编译以后，最后也是转换成汇编码的。

一条指令的执行是可以分为很多步骤的，假设cpu指令分为以下几步

- 取指 IF
- 译码和取寄存器操作数 ID
- 执行或者有效地址计算 EX
- 存储器访问 MEM
- 写回 WB

假设这里有两条指令

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_2.jpg)

一般来说我们会认为指令是串行执行的，先执行指令1，然后再执行指令2。假设每个步骤需要消耗1个cpu时间周期，那么执行这两个指令需要消耗10个cpu时间周期，这样做效率太低。事实上指令都是并行执行的，当然在第一条指令在执行IF的时候，第二条指令是不能进行IF的，因为指令寄存器等不能被同时占用。所以就如上图所示，两条指令是一种相对错开的方式并行执行。当指令1执行ID的时候，指令2执行IF。这样只用6个cpu时间周期就执行了两个指令，效率比较高。

按照这个思路我们来看下A=B+C的指令是如何执行的。

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_3.jpg)

如图所示，ADD操作时有一个空闲（X）操作，因为当想让B和C相加的时候，在图中ADD的X操作时，C还没从内存中读取（当MEM操作完成时，C才从内存中读取。这里会有一个疑问，此时还没有回写（WB）到R2中，怎么会将R1与R1相加。那是因为在硬件电路当中，会使用一种叫“旁路”的技术直接把数据从硬件当中读取出来，所以不需要等待WB执行完才进行ADD）。所以ADD操作中会有一个空闲（X）时间。在SW操作中，因为EX指令不能和ADD的EX指令同时进行，所以也会有一个空闲（X）时间。

接下来举个稍微复杂点的例子

```java
a = b + c 
d = e - f
```

对应的指令如下图

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_4.jpg)

原因和上面的类似，这里就不分析了。我们发现，这里的X很多，浪费的时间周期很多，性能也被影响。有没有办法使X的数量减少呢？

我们希望用一些操作把X的空闲时间填充掉，因为ADD与上面的指令有数据依赖，我们希望用一些没有数据依赖的指令去填充掉这些因为数据依赖而产生的空闲时间。

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_5.jpg)

我们将指令的顺序进行了改变

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_6.jpg)

改变了指令顺序以后，X被消除了。总体的运行时间周期也减少了。

**指令重排可以使流水线更加顺畅**

当然指令重排的原则是不能破坏串行程序的语义，例如a=1,b=a+1，这种指令就不会重排了，因为重排的串行结果和原先的不同。

指令重排只是编译器或者CPU的优化一种方式，而这种优化就造成了本章一开始程序的问题。

如何解决呢？用volatile关键字，这个后面的系列会介绍到。



## 3. 可见性

- 可见性是指当一个线程修改了某一个共享变量的值，其他线程是否能够立即知道这个修改。

可见性问题可能有各个环节产生。比如刚刚说的指令重排也会产生可见性问题，另外在编译器的优化或者某些硬件的优化都会产生可见性问题。

比如某个线程将一个共享值优化到了内存中，而另一个线程将这个共享值优化到了缓存中，当修改内存中值的时候，缓存中的值是不知道这个修改的。

比如有些硬件优化，程序在对同一个地址进行多次写时，它会认为是没有必要的，只保留最后一次写，那么之前写的数据在其他线程中就不可见了。

总之，可见性的问题大多都源于优化。

接下来看一个Java虚拟机层面产生的可见性问题

```java
package edu.hushi.jvm;
 
/**
 *
 * @author -10
 *
 */
public class VisibilityTest extends Thread {
 
    private boolean stop;
 
    public void run() {
        int i = 0;
        while(!stop) {
            i++;
        }
        System.out.println("finish loop,i=" + i);
    }
 
    public void stopIt() {
        stop = true;
    }
 
    public boolean getStop(){
        return stop;
    }
    public static void main(String[] args) throws Exception {
        VisibilityTest v = new VisibilityTest();
        v.start();
 
        Thread.sleep(1000);
        v.stopIt();
        Thread.sleep(2000);
        System.out.println("finish main");
        System.out.println(v.getStop());
    }
 
}
```

代码很简单，v线程一直不断的在while循环中i++，直到主线程调用stop方法，改变了v线程中的stop变量的值使循环停止。

看似简单的代码运行时就会出现问题。这个程序在 client 模式下是能停止线程做自增操作的，但是在 **server 模式先将是无限循环**。（server模式下JVM优化更多）

64位的系统上面大多都是server模式，在server模式下运行：

```java
finish main
true
```

只会打印出这两句话，而不会打印出finish loop。可是能够发现stop的值已经是true了。

该Blog作者用工具将程序还原为汇编代码

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_7.jpg)

这里只截取了一部分汇编代码，红色部分为循环部分，可以清楚得看到只有在0x0193bf9d才进行了stop的验证，而红色部分并没有取stop的值，所以才进行了无限循环。

这是JVM优化后的结果。如何避免呢？和指令重排一样，用volatile关键字。

如果加入了volatile，再还原为汇编代码就会发现，每次循环都会get一下stop的值。

接下来看一些在“Java语言规范”中的示例

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_8.jpg)

上图说明了指令重排将会导致结果不同。

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_9.jpg)

上图使r5=r2的原因是，r2=r1.x，r5=r1.x，在编译时直接将其优化成r5=r2。最后导致结果不同。



## 4. Happen-Before

- 程序顺序原则：一个线程内保证语义的串行性
- volatile规则：volatile变量的写，先发生于读，这保证了volatile变量的可见性
- 锁规则：解锁（unlock）必然发生在随后的加锁（lock）前
- 传递性：A先于B，B先于C，那么A必然先于C
- 线程的start()方法先于它的每一个动作
- 线程的所有操作先于线程的终结（Thread.join()）
- 线程的中断（interrupt()）先于被中断线程的代码
- 对象的构造函数执行结束先于finalize()方法

这些原则保证了重排的语义是一致的。详见 [Happen-before & DCL](https://github.com/WhiteSmithFloyd/Java_Basic/blob/master/Threadful%20High/Happen-before%20%26%20DCL.md)



## 5. 线程安全的概念

指某个函数、函数库在多线程环境中被调用时，能够正确地处理各个线程的局部变量，使程序功能正确完成。

比如最开始所说的i++的例子

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/high_concurrent/hc_3_10.jpg)

就会导致线程不安全。






