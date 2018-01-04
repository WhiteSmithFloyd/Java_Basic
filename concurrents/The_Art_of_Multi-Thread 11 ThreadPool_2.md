# The Art of Multi-Thread 11 线程池二

## Executor两级调度模型
![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/art_thread/art_thread_11_1.jpg)     
在HotSpot虚拟机中，Java中的线程将会被一一映射为操作系统的线程。 
在Java虚拟机层面，用户将多个任务提交给Executor框架，Executor负责分配线程执行它们； 
在操作系统层面，操作系统再将这些线程分配给处理器执行。


*** 

## Executor结构
![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/art_thread/art_thread_11_2.jpg)

Executor框架中的所有类可以分成三类：
+ 任务 
任务有两种类型：Runnable和Callable。   

+ 任务执行器 
Executor框架最核心的接口是Executor，它表示任务的执行器。 
Executor的子接口为ExecutorService。 
ExecutorService有两大实现类：ThreadPoolExecutor和ScheduledThreadPoolExecutor。

+ 执行结果 
Future接口表示异步的执行结果，它的实现类为FutureTask。


## 线程池
Executors工厂类可以创建四种类型的线程池，通过Executors.newXXX即可创建。

### 1. FixedThreadPool
```java
public static ExecutorService newFixedThreadPool(int nThreads){
    return new ThreadPoolExecutor(nThreads,nThreads,0L,TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());
}
```
![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/art_thread/art_thread_11_3.jpg)     
+ 它是一种固定大小的线程池；
+ corePoolSize和maximunPoolSize都为用户设定的线程数量nThreads；
+ keepAliveTime为0，意味着一旦有多余的空闲线程，就会被立即停止掉；但这里keepAliveTime无效；
+ 阻塞队列采用了LinkedBlockingQueue，它是一个无界队列；
+ 由于阻塞队列是一个无界队列，因此永远不可能拒绝任务；
+ 由于采用了无界队列，实际线程数量将永远维持在nThreads，因此maximumPoolSize和keepAliveTime将无效。


### 2. CachedThreadPool
```java
public static ExecutorService newCachedThreadPool(){
    return new ThreadPoolExecutor(0,Integer.MAX_VALUE,60L,TimeUnit.MILLISECONDS,new SynchronousQueue<Runnable>());
}
```
![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/art_thread/art_thread_11_4.jpg)      
+ 它是一个可以无限扩大的线程池；
+ 它比较适合处理执行时间比较小的任务；
+ corePoolSize为0，maximumPoolSize为无限大，意味着线程数量可以无限大；
+ keepAliveTime为60S，意味着线程空闲时间超过60S就会被杀死；
+ 采用SynchronousQueue装等待的任务，这个阻塞队列没有存储空间，这意味着只要有请求到来，就必须要找到一条工作线程处理他，如果当前没有空闲的线程，那么就会再创建一条新的线程。


### 3. SingleThreadExecutor
```java
public static ExecutorService newSingleThreadExecutor(){
    return new ThreadPoolExecutor(1,1,0L,TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());
}
```
![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/art_thread/art_thread_11_5.jpg)       
+ 它只会创建一条工作线程处理任务；
+ 采用的阻塞队列为LinkedBlockingQueue；


###  4. ScheduledThreadPool
它用来处理延时任务或定时任务。   

![Image](https://github.com/WhiteSmithFloyd/ress/blob/master/imgs/art_thread/art_thread_11_6.jpg) 


+ 它接收SchduledFutureTask类型的任务，有两种提交任务的方式：
  - scheduledAtFixedRate
  - scheduledWithFixedDelay
  
+ SchduledFutureTask接收的参数：
  - time：任务开始的时间
  - sequenceNumber：任务的序号
  - period：任务执行的时间间隔
  
+ 它采用[DelayQueue](https://github.com/WhiteSmithFloyd/Java_Basic/blob/master/concurrents/DelayQueue.md)存储等待的任务
  - DelayQueue内部封装了一个PriorityQueue，它会根据time的先后时间排序，若time相同则根据sequenceNumber排序；
  - DelayQueue也是一个无界队列；

+ 工作线程的执行过程：
  - 工作线程会从DelayQueue取已经到期的任务去执行；
  - 执行结束后重新设置任务的到期时间，再次放回DelayQueue




