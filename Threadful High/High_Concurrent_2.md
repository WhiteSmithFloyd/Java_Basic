# 高并发 - 多线程基础 [From](https://my.oschina.net/hosee/blog/599000)

## 1、什么是线程

线程是进程内的执行单元



某个进程当中都有若干个线程。

线程是进程内的执行单元。

使用线程的原因是，进程的切换是非常重量级的操作，非常消耗资源。如果使用多进程，那么并发数相对来说不会很高。而线程是更细小的调度单元，更加轻量级，所以线程会较为广泛的用于并发设计。

在Java当中线程的概念和操作系统级别线程的概念是类似的。事实上，Jvm将会把Java中的线程映射到操作系统的线程区。

## 2. 线程的基本操作

### 2.1 线程状态图



上图是Java中线程的基本操作。

当new出一个线程时，其实线程并没有工作。它只是生成了一个实体，当你调用这个实例的start方法时，线程才真正地被启动。启动后到Runnable状态，Runnable表示该线程的资源等等已经被准备好，已经可以执行了，但是并不表示一定在执行状态，由于时间片轮转，该线程也可能此时并没有在执行。对于我们来说，该线程可以认为已经被执行了，但是是否真实执行，还得看物理cpu的调度。当线程任务执行结束后，线程就到了Terminated状态。

- 有时候在线程的执行当中，不可避免的会申请某些锁或某个对象的监视器，当无法获取时，这个线程会被阻塞住，会被挂起，到了Blocked状态。
- 如果这个线程调用了wait方法，它就处于一个Waiting状态。进入Waiting状态的线程会等待其他线程给它notify，通知到之后由Waiting状态又切换到Runnable状态继续执行。
- 当然等待状态有两种，一种是无限期等待，直到被notify。一直则是有限期等待，比如等待10秒还是没有被notify，则自动切换到Runnable状态。

### 2.2 新建线程

```java
Thread thread = new Thread();
thread.start();
```

这样就开启了一个线程。

run方法的实现有两种方式

第一种方式，直接覆盖run方法，就如刚刚代码中所示，最方便的用一个匿名类就可以实现。

```java
Thread thread = new Thread("t1")
{
	@Override
	public void run()
	{
		// TODO Auto-generated method stub
		System.out.println(Thread.currentThread().getName());
	}
};
```

第二种方式

```java
Thread t1=new Thread(new CreateThread3());
```

CreateThread3()实现了Runnable接口。

### 2.3 终止线程

+ Thread.stop() 不推荐使用。它会释放所有monitor

在源码中已经明确说明stop方法被Deprecated，在Javadoc中也说明了原因。
原因在于stop方法太过"暴力"了，无论线程执行到哪里，它将会立即停止掉线程。

![Image]()



当写线程得到锁以后开始写入数据，写完id = 1，在准备将name = 1时被stop,释放锁。读线程获得锁进行读操作，读到的id为1，而name还是0，导致了数据不一致。

最重要的是这种错误不会抛出异常，将很难被发现。



### 2.4 线程中断

线程中断有3种方法

```java
public void Thread.interrupt() // 中断线程 
public boolean Thread.isInterrupted() // 判断是否被中断 
public static boolean Thread.interrupted() // 判断是否被中断，并清除当前中断状态
```

什么是线程中断呢？

如果不了解Java的中断机制，这样的一种解释极容易造成误解，认为调用了线程的interrupt方法就一定会中断线程。 
其实，Java的中断是一种协作机制。也就是说调用线程对象的interrupt方法并不一定就中断了正在运行的线程，它只是要求线程自己在合适的时机中断自己。每个线程都有一个boolean的中断状态（不一定就是对象的属性，事实上，该状态也确实不是Thread的字段）**，interrupt方法仅仅只是将该状态置为true**。对于非阻塞中的线程, 只是改变了中断状态, 即Thread.isInterrupted()将返回true，并不会使程序停止; 

```java
public void run(){//线程t1
   while(true){
      Thread.yield();
   }
}
t1.interrupt();
```

这样使线程t1中断，是不会有效果的，只是更改了中断状态位。

如果希望非常优雅地终止这个线程，就该这样做

```java
public void run(){ 
    while(true)
    { 
        if(Thread.currentThread().isInterrupted())
        { 
           System.out.println("Interruted!"); 
           break; 
        } 
        Thread.yield(); 
    } 
}
```

使用中断，就对数据一致性有了一定的保证。



对于可取消的阻塞状态中的线程, 比如等待在这些函数上的线程, Thread.sleep(), Object.wait(), Thread.join(), 这个线程收到中断信号后, 会抛出InterruptedException, 同时会把中断状态置回为false.

对于取消阻塞状态中的线程，可以这样抒写代码：

```java
public void run(){
    while(true){
        if(Thread.currentThread().isInterrupted()){
            System.out.println("Interruted!");
            break;
        }
        try {
           Thread.sleep(2000);
        } catch (InterruptedException e) {
           System.out.println("Interruted When Sleep");
           //设置中断状态，抛出异常后会清除中断标记位
           Thread.currentThread().interrupt();
        }
        Thread.yield();
    }
}
```



### 2.5 线程挂起

挂起（suspend）和继续执行（resume）线程

+ suspend()不会释放锁
+ 如果加锁发生在resume()之前 ，则死锁发生

这两个方法都是Deprecated方法，不推荐使用。



### 2.6 join和yeild

yeild是个native静态方法，这个方法是想把自己占有的cpu时间释放掉，然后和其他线程一起竞争(注意yeild的线程还是有可能争夺到cpu，注意与sleep区别)。在javadoc中也说明了，yeild是个基本不会用到的方法，一般在debug和test中使用。

join方法的意思是等待其他线程结束，就如suspend那节的代码，想让主线程等待t1,t2结束以后再结束。没有结束的话，主线程就一直阻塞在那里。

```java
package test;

public class Test
{
	public volatile static int i = 0;

	public static class AddThread extends Thread
	{
		@Override
		public void run()
		{
			for (i = 0; i < 10000000; i++)
				;
		}
	}

	public static void main(String[] args) throws InterruptedException
	{
		AddThread at = new AddThread();
		at.start();
		at.join();
		System.out.println(i);
	}
}
```

如果把上述代码的at.join去掉，则主线程会直接运行结束，i的值会很小。如果有join,打印出的i的值一定是10000000。



那么join是怎么实现的呢？

join的本质 

```java
while(isAlive()) 
{ 
   wait(0); 
}
```

join()方法也可以传递一个时间，意为有限期地等待，超过了这个时间就自动唤醒。

这样就有一个问题，谁来notify这个线程呢，在thread类中没有地方调用了notify？

在javadoc中，找到了相关解释。当一个线程运行完成终止后，将会调用notifyAll方法去唤醒等待在当前线程实例上的所有线程,这个操作是jvm自己完成的。

所以javadoc中还给了我们一个建议，不要使用wait和notify/notifyall在线程实例上。因为jvm会自己调用，有可能与你调用期望的结果不同。



## 3. 守护线程

- 在后台默默地完成一些系统性的服务，比如垃圾回收线程、JIT线程就可以理解为守护线程。
- 当一个Java应用内，所有非守护进程都结束时，Java虚拟机就会自然退出。

```java
Thread t=new DaemonT(); 
t.setDaemon(true); 
t.start();
```

这样就开启了一个守护线程。

```java
package test;

public class Test
{
	public static class DaemonThread extends Thread
	{
		@Override
		public void run()
		{
			for (int i = 0; i < 10000000; i++)
			{
				System.out.println("hi");
			}
		}
	}

	public static void main(String[] args) throws InterruptedException
	{
		DaemonThread dt = new DaemonThread();
		dt.start();
	}
}
```

当线程dt不是一个守护线程时，在运行后，我们能看到控制台输出hi

当在start之前加入

```java
dt.setDaemon(true);
```

控制台就直接退出了，并没有输出。



## 4. 线程优先级

Thread类中有3个变量定义了线程优先级。

```java
public final static int MIN_PRIORITY = 1;
public final static int NORM_PRIORITY = 5;
public final static int MAX_PRIORITY = 10;
```

```java
package test;

public class Test
{
	public static class High extends Thread
	{
		static int count = 0;
		@Override
		public void run()
		{
			while (true)
			{
				synchronized (Test.class)
				{
					count++;
					if (count > 10000000)
					{
						System.out.println("High");
						break;
					}
				}
			}
		}
	}
	public static class Low extends Thread
	{
		static int count = 0;
		@Override
		public void run()
		{
			while (true)
			{
				synchronized (Test.class)
				{
					count++;
					if (count > 10000000)
					{
						System.out.println("Low");
						break;
					}
				}
			}
		}
	}

	public static void main(String[] args) throws InterruptedException
	{
		High high = new High();
		Low low = new Low();
		high.setPriority(Thread.MAX_PRIORITY);
		low.setPriority(Thread.MIN_PRIORITY);
		low.start();
		high.start();
	}
}
```

让一个高优先级的线程和低优先级的线程同时争夺一个锁，看看哪个最先完成。

**当然并不一定是高优先级一定先完成。再多次运行后发现，高优先级完成的概率比较大，但是低优先级还是有可能先完成的**。



## 5. 基本的线程同步操作

### synchronized 和 Object.wait() Obejct.notify()

主要要注意的是

synchronized有三种加锁方式：

- 指定加锁对象：对给定对象加锁，进入同步代码前要获得给定对象的锁。
- 直接作用于实例方法：相当于对当前实例加锁，进入同步代码前要获得当前实例的锁。
- 直接作用于静态方法：相当于对当前类加锁，进入同步代码前要获得当前类的锁。

作用于实例方法，则不要new两个不同的实例

作用于静态方法，只要类一样就可以了，因为加的锁是类.class，可以new两个不同实例。

wait和notify的用法：

用什么锁住，就用什么调用wait和notify



