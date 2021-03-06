# The Art of Multi-Thread 8 Executors

# Executors
Executor框架便是Java 5中引入的，其内部使用了线程池机制，它在java.util.cocurrent 包下，通过该框架来控制线程的启动、执行和关闭，可以简化并发编程的操作。因此，在Java 5之后，通过Executor来启动线程比使用Thread的start方法更好，除了更易管理，效率更好（用线程池实现，节约开销）外，还有关键的一点：有助于避免[this逃逸](http://coolxing.iteye.com/blog/1464501)

Executor框架包括：线程池，Executor，Executors，ExecutorService，CompletionService，Future，Callable等。



## Executor类
Executor接口中之定义了一个方法execute(Runnable command)，该方法接收一个Runable实例，它用来执行一个任务，任务即一个实现了Runnable接口的类。



## ExecutorService类
ExecutorService接口继承自Executor接口，它提供了更丰富的实现多线程的方法，比如，ExecutorService提供了关闭自己的方法，以及可为跟踪一个或多个异步任务执行状况而生成 Future 的方法。

ExecutorService的生命周期包括三种状态：**运行**、**关闭**、**终止**。创建后便进入运行状态，当调用了shutdown()方法时，便进入关闭状态，此时意味着ExecutorService不再接受新的任务，但它还在执行已经提交了的任务，当所有已经提交了的任务执行完后，便到达终止状态。
如果不调用shutdown()方法，ExecutorService会一直处在运行状态，不断接收新的任务，执行新的任务，服务器端一般不需要关闭它，保持一直运行即可。



## Executors类
Executors提供了一系列工厂方法用于创先线程池，返回的线程池都实现了ExecutorService接口。






### public static ExecutorService newFixedThreadPool(int nThreads)
+ 创建固定数目线程的线程池。
+ newFixedThreadPool与cacheThreadPool差不多，也是能reuse就用，但不能随时建新的线程；
+ 任意时间点，最多只能有固定数目的活动线程存在，此时如果有新的线程要建立，只能放在另外的队列中等待，直到当前的线程中某个线程终止直接被移出池子；
+ 和cacheThreadPool不同，FixedThreadPool没有IDLE机制（可能也有，但既然文档没提，肯定非常长，类似依赖上层的TCP或UDP IDLE机制之类的），所以FixedThreadPool多数针对一些很稳定很固定的正规并发线程，多用于服务器；
+ 从方法的源代码看，cache池和fixed 池调用的是同一个底层 池，只不过参数不同: 
fixed池线程数固定，并且是0秒IDLE（无IDLE） 
cache池线程数支持0-Integer.MAX_VALUE(显然完全没考虑主机的资源承受能力），60秒IDLE。






***
### public static ExecutorService newCachedThreadPool()
+ 创建一个可缓存的线程池，调用execute将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线程并添加到池中。终止并从缓存中移除那些已有60秒钟未被使用的线程。
+ 缓存型池子通常用于执行一些生存期很短的异步型任务；
+ 注意，放入CachedThreadPool的线程不必担心其结束，超过TIMEOUT不活动，其会自动被终止。
+ 缺省timeout是60s。



***
### public static ExecutorService newSingleThreadExecutor()
+ 创建一个单线程化的Executor。
+ 用的是和cache池和fixed池相同的底层池，但线程数目是1-1,0秒IDLE（无IDLE）



***
### public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) 
创建一个支持定时及周期性的任务执行的线程池，多数情况下可用来替代Timer类.


> #### Timer类存在以下缺陷：
> + Timer类不管启动多少定时器，但它只会启动一条线程，当有多个定时任务时，就会产生延迟。如：我们要求一个任务每隔3S执行，且执行大约需要10S，第二个任务每隔5S执行，两个任务同时启动。若使用Timer我们会发现，第而个任务是在第一个任务执行结束后的5S才开始执行。这就是多任务的延时问题。
> + 若多个定时任务中有一个任务抛异常，那所有任务都无法执行。
> + Timer执行周期任务时依赖系统时间。若系统时间发生变化，那Timer执行结果可能也会发生变化。而ScheduledExecutorService基于时间的延迟，并非时间，因此不会由于系统时间的改变发生执行变化。 
综上所述，定时任务要使用ScheduledExecutorService取代Timer。



## Executor执行任务
在Java 5之后，任务分两类：一类是实现了**Runnable接口**的类，一类是实现了**Callable接口**的类。      
两者都可以被ExecutorService执行，但是Runnable任务没有返回值，而Callable任务有返回值。并且Callable的call()方法只能通过ExecutorService的submit(Callable task) 方法来执行，并且返回一个Future，是表示任务等待完成的Future。

> Callable接口类似于Runnable，两者都是为那些其实例可能被另一个线程执行的类设计的。但是 Runnable 不会返回结果，并且无法抛出经过检查的异常而Callable又返回结果，而且当获取返回结果时可能会抛出异常。Callable中的call()方法类似Runnable的run()方法，区别同样是有返回值，后者没有。

当将一个**Callable的对象**传递给**ExecutorService的submit方法**，则该call方法自动在一个线程上执行，并且会返回执行结果Future对象。     
同样，将**Runnable的对象**传递给**ExecutorService的submit方法**，则该run方法自动在一个线程上执行，并且会返回执行结果Future对象，但是在该Future对象上调用get方法，将返回null。





### Executor执行Runnable任务
通过Executors的以上四个静态工厂方法获得 ExecutorService实例，而后调用该实例的execute（Runnable command）方法即可。一旦Runnable任务传递到execute（）方法，该方法便会自动在一个线程上执行。
```java
// 获取ExecutorService实例
ExecutorService executorService = Executors.newCachedThreadPool();

// 提交任务
executorService.execute( new Runnable(){
    public void run(){
        //……
    }
} );
```

### Executor执行Callable任务
```java
// 创建线程池
ExecutorService executorService = Executors.newCachedThreadPool();

// 提交任务
Future<String> future = executorService.submit( new Callable<String>{
    public String call(){
        // ……
    }
} );

// 获取执行结果
if ( future.isDone ) {
    String result = future.get();
}

// 关闭线程池
executorService.shutdown();
```
+ Callable<String>表示call函数返回值为String类型；
+ 如果Future的返回尚未完成，则 **get()方法会阻塞等待** ，直到Future完成返回，可以通过调用isDone()方法判断Future是否完成了返回。



******

## ThreadPoolExecutor类
该类用于构造自定义的线程池。构造方法如下：
```java
public ThreadPoolExecutor (int corePoolSize, int maximumPoolSize, long keepAliveTime,
  TimeUnit unit,BlockingQueue<Runnable> workQueue)
```
+ corePoolSize：线程池中所保存的核心线程数，包括空闲线程。线程池认为这是一个最合理的值，它会尽量使得线程数量维持在这个值上下。
+ maximumPoolSize：池中允许的最大线程数。
+ keepAliveTime：线程池中的空闲线程所能持续的最长时间。
+ unit：持续时间的单位。
+ workQueue：任务执行前保存任务的队列，仅保存由execute方法提交的Runnable任务。


当试图通过excute方法讲一个Runnable任务添加到线程池中时，按照如下顺序来处理：
1. 如果线程池中的线程数量少于corePoolSize，即使线程池中有空闲线程，也会创建一个新的线程来执行新添加的任务；
2. 如果线程池中的线程数量大于等于corePoolSize，但缓冲队列workQueue未满，则不再创建新的线程，并将新任务放到workQueue中，按照FIFO的原则依次等待执行（线程池中有线程空闲出来后依次将缓冲队列中的任务交付给空闲的线程执行）；
3. 如果线程池中的线程数量大于等于corePoolSize，且缓冲队列workQueue已满，但线程池中的线程数量小于maximumPoolSize，则会创建新的线程来处理被添加的任务；
4. 如果线程池中的线程数量等于了maximumPoolSize，有4种才处理方式（该构造方法调用了含有5个参数的构造方法，并将最后一个构造方法为RejectedExecutionHandler类型，它在处理线程溢出时有4种方式，这里不再细说，要了解的，自己可以阅读下源码）。
5. 另外，当线程池中的线程数量大于corePoolSize时，如果里面有线程的空闲时间超过了keepAliveTime，就将其移除线程池，这样，可以动态地调整线程池中线程的数量。





