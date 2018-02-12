# JVM GC 1

## 1. GC的概念

Garbage Collection 垃圾收集

Java中，GC的对象是堆空间和永久区(如果对Java内存区域不太了解，请查看[Java内存区域](https://github.com/WhiteSmithFloyd/Java_Basic/blob/master/JVMs/Java%20Memory%20Mode.md))



## 2. GC算法

### 2.1 引用计数法

老牌垃圾回收算法
通过引用计算来回收垃圾

使用者：

- COM
- ActionScript3
- Python

引用计数器的实现很简单，对于一个对象A，只要有任何一个对象引用了A，则A的引用计数器就加1，当引用失效时，引用计数器就减1。只要对象A的引用计数器的值为0，则对象A就不可能再被使用。

所以当一个对象的引用数量为0，就意味着没有人引用了这个对象，就可以将这个对象GC掉。

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_1_1.jpg)



**引用计数法的问题**：

- 引用和去引用伴随加法和减法，影响性能
- 很难处理循环引用

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_1_2.jpg)

从上图可以看出，右边的3个对象，最后的引用计数都是1，但是都不被根对象所引用，三个垃圾对象循环引用，导致都无法被回收。

这里要注意的是，引用计数法在Java中没有采用。



### 2.2 标记-清除

标记-清除算法是现代垃圾回收算法的思想基础。标记-清除算法将垃圾回收分为两个阶段：标记阶段和清除阶段。一种可行的实现是，在标记阶段，首先通过根节点，标记所有从根节点开始的可达对象。因此，未被标记的对象就是未被引用的垃圾对象。然后，在清除阶段，清除所有未被标记的对象。 



### 2.3 标记-压缩

标记-压缩算法适合用于存活对象较多的场合，如老年代。它在标记-清除算法的基础上做了一些优化。和标记-清除算法一样，标记-压缩算法也首先需要从根节点开始，对所有可达对象做一次标记。但之后，它并不简单的清理未标记的对象，而是将所有的存活对象压缩到内存的一端。之后，清理边界外所有的空间。 

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_1_3.jpg)





### 2.4 复制算法

- 与标记-清除算法相比，复制算法是一种相对高效的回收方法
- 不适用于存活对象较多的场合 如老年代
- 将原有的内存空间分为两块，每次只使用其中一块，在垃圾回收时，将正在使用的内存中的存活对象复制到未使用的内存块中，之后，清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_1_4.jpg)



**复制算法的问题**：空间浪费，只能使用一半空间

通过整合标记清理思想来使得空间不怎么浪费

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_1_5.jpg)

上图中老年代与复制算法关系不大，老年代一般是担保空间。
最上面那块大的区域，当做对象产生的空间。

中间那块就是复制算法的核心区域，两块区域相同大小的空间。

具体步骤是：

1. 在最上面那块大的区域产生新对象。
2. 大对象不太适合在复制空间，因为复制空间的容量是有限的，所以需要一个大的空间做担保，所以让老年代做担保。这样产生的大对象直接进入老年代。
3. 每一次GC，对象的年龄就会+1，一个对象在几次GC后仍然没有被回收，则这个对象就是一个老年对象。老年对象是一个长期被引用的对象，老年对象将被放入老年代。
4. 步骤1中产生的小对象，将进入到复制空间。原先复制空间中的新对象也将被复制到另一块复制空间
5. 清空垃圾对象

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/jvms/jvm_gc_1_6.jpg)

根据-XX:+PrintGCDetails的输出，我们可以看到

一个堆分为new generation(新生代) , tenured generation(老年代)和compacting perm gen。

而new generation分为eden space,from space（有些地方称为s0和s1，表示幸存代） , to space。

eden space就是上面那种图中，对象产生的地方。

from space和to space是两块大小一样的区域，是上图中的复制空间。

new generation的可用总空间就是eden space+一块复制空间（另一块不算），但是根据new generation的地址访问可以算出是eden space + 两块复制空间区域，所以复制算法浪费了一部分空间。





### 2.5 分代思想

依据对象的存活周期进行分类，短命对象归为新生代，长命对象归为老年代。
根据不同代的特点，选取合适的收集算法

- 少量对象存活，适合复制算法
- 大量对象存活，适合标记清理或者标记压缩



进入老年代的对象有两种情况：

1. 新生代空间不够，老年代做担保存放一些大对象
2. 某些对象多次GC后仍然存在，进入老年代。



老年代的大多数对象都是第2种情况，所以老年代的对象的生命周期比较长，GC的发生也比较少，会有大量对象存活，所以不用复制算法，而改为标记清理或者标记压缩。 

**所有的算法，需要能够识别一个垃圾对象，因此需要给出一个可触及性的定义**



## 3. 可触及性

**可触及的**

- 从根节点可以触及到这个对象

**可复活的** 

- 一旦所有引用被释放，就是可复活状态
- 因为在finalize()中可能复活该对象

**不可触及的** 

- 在finalize()后，可能会进入不可触及状态
- 不可触及的对象不可能复活
- 可以回收

下面举个例子来说明可复活这个状态：

```java
public class CanReliveObj
{
	public static CanReliveObj obj;

	@Override
	protected void finalize() throws Throwable
	{
		super.finalize();
		System.out.println("CanReliveObj finalize called");
		obj = this;
	}

	@Override
	public String toString()
	{
		return "I am CanReliveObj";
	}

	public static void main(String[] args) throws InterruptedException
	{
		obj = new CanReliveObj();
		obj = null; // 可复活
		System.gc();
		Thread.sleep(1000);
		if (obj == null)
		{
			System.out.println("obj 是 null");
		}
		else
		{
			System.out.println("obj 可用");
		}
		System.out.println("第二次gc");
		obj = null; // 不可复活
		System.gc();
		Thread.sleep(1000);
		if (obj == null)
		{
			System.out.println("obj 是 null");
		}
		else
		{
			System.out.println("obj 可用");
		}
	}

}
```

输出：

```tex
CanReliveObj finalize called
obj 可用
第二次gc
obj 是 null
```

一般我们认为，对象赋值null后，对象就可以被GC了，在上述实例中，在finalize中，又将obj=this，使对象复活。因为finalize只能调用一次，所以第二次GC时，obj被回收。



因此对于finalize会有这样的建议：

- 经验：避免使用finalize()，操作不慎可能导致错误。
- finalize优先级低，何时被调用（在GC时被调用，何时发生GC不确定） 不确定
- 可以使用try-catch-finally来替代它

另外在之前，我们一直在提到从根出发，那么根是指哪些对象呢？

- 栈中引用的对象
- 方法区中静态成员或者常量引用的对象（全局对象）
- JNI方法栈中引用对象





## 4. Stop-The-World

Stop-The-World是Java中一种全局暂停的现象。

全局停顿，所有Java代码停止，native代码可以执行，但不能和JVM交互

多半由于GC引起，当然Dump线程、死锁检查、堆Dump都有可能引起Stop-The-World



**GC时为什么会有全局停顿？**

类比在聚会时打扫房间，聚会时很乱，又有新的垃圾产生，房间永远打扫不干净，只有让大家停止活动了，才能将房间打扫干净。

危害

长时间服务停止，没有响应
遇到HA系统，可能引起主备切换，严重危害生产环境。

**新生代的GC（****Minor GC********），停顿时间比较短**

**老年代的GC（Major GC），停顿时间可能比较长**




