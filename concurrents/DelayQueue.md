# DelayQueue

## 基本简介

在谈到DelayQueue的使用和原理的时候，我们首先介绍一下DelayQueue。
DelayQueue是一个**无界阻塞队列**，只有在**延迟期满**时才能从中提取元素。
**该队列的头部**是延迟期满后**保存时间最长**的Delayed 元素。



## DelayQueue使用场景

a) 关闭空闲连接。服务器中，有很多客户端的连接，空闲一段时间之后需要关闭之。
b) 缓存。缓存中的对象，超过了空闲时间，需要从缓存中移出。
c) 任务超时处理。在网络协议滑动窗口请求应答式交互时，处理超时未响应的请求。

如果不使用DelayQueue，那么常规的解决办法就是：使用一个后台线程，遍历所有对象，挨个检查。这种笨笨的办法简单好用，但是对象数量过多时，可能存在性能问题，检查间隔时间不好设置，间隔时间过大，影响精确度，过小则存在效率问题。而且做不到按超时的时间顺序处理。



## DelayQueue应用示例

为了具有调用行为，存放到DelayDeque的元素必须继承**Delayed接口**。Delayed接口使对象成为延迟对象，它使存放在DelayQueue类中的对象具有了激活日期。该接口强制执行下列两个方法。

- **CompareTo(Delayed o)**: Delayed接口继承了Comparable接口，因此有了这个方法。
- **getDelay(TimeUnit unit)**:  这个方法返回到激活日期的剩余时间，时间单位由单位参数指定。



```java
public class DelayEvent implements Delayed {
    private Date startDate;
    public DelayEvent(Date startDate) {
        super();
        this.startDate = startDate;
    }
    @Override
    public int compareTo(Delayed o) {
        long result = this.getDelay(TimeUnit.NANOSECONDS)
                - o.getDelay(TimeUnit.NANOSECONDS);
        if (result < 0) {
            return -1;
        } else if (result > 0) {
            return 1;
        } else {
            return 0;
        }
    }
    @Override
    public long getDelay(TimeUnit unit) {
        Date now = new Date();
        long diff = startDate.getTime() - now.getTime();
        return unit.convert(diff, TimeUnit.MILLISECONDS);
    }
}

// 
public class DelayTask implements Runnable {
    private int id;
    private DelayQueue<DelayEvent> queue;
    public DelayTask(int id, DelayQueue<DelayEvent> queue) {
        super();
        this.id = id;
        this.queue = queue;
    }
    @Override
    public void run() {
        Date now = new Date();
        Date delay = new Date();
        delay.setTime(now.getTime() + id * 1000);
        System.out.println("Thread " + id + " " + delay);
        for (int i = 0; i < 100; i++) {
            DelayEvent delayEvent = new DelayEvent(delay);
            queue.add(delayEvent);
        }
    }
}

//
public class DelayDequeMain {
    public static void main(String[] args) throws Exception {
        DelayQueue<DelayEvent> queue = new DelayQueue<DelayEvent>();
        Thread threads[] = new Thread[5];
        for (int i = 0; i < threads.length; i++) {
            DelayTask task = new DelayTask(i + 1, queue);
            threads[i] = new Thread(task);
        }
        for (int i = 0; i < threads.length; i++) {
            threads[i].start();
        }
        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        do {
            int counter = 0;
            DelayEvent delayEvent;
            do {
                delayEvent = queue.poll();
                if (delayEvent != null) {
                    counter++;
                }
            } while (delayEvent != null);
            System.out.println("At " + new Date() + " you have read " + counter+ " event");
            TimeUnit.MILLISECONDS.sleep(500);
        } while (queue.size() > 0);
    }
}

```





result:

```tex
Thread 3 Fri May 06 11:00:20 CST 2016
Thread 1 Fri May 06 11:00:18 CST 2016
Thread 5 Fri May 06 11:00:22 CST 2016
Thread 4 Fri May 06 11:00:21 CST 2016
Thread 2 Fri May 06 11:00:19 CST 2016
At Fri May 06 11:00:17 CST 2016 you have read 0 event
At Fri May 06 11:00:18 CST 2016 you have read 0 event
At Fri May 06 11:00:18 CST 2016 you have read 100 event
At Fri May 06 11:00:19 CST 2016 you have read 0 event
At Fri May 06 11:00:19 CST 2016 you have read 100 event
At Fri May 06 11:00:20 CST 2016 you have read 0 event
At Fri May 06 11:00:20 CST 2016 you have read 100 event
At Fri May 06 11:00:21 CST 2016 you have read 0 event
At Fri May 06 11:00:21 CST 2016 you have read 100 event
At Fri May 06 11:00:22 CST 2016 you have read 0 event
At Fri May 06 11:00:22 CST 2016 you have read 100 event
```





## DelayQueue基本原理

首先，这种队列中只能存放实现Delayed接口的对象，而此接口有两个需要实现的方法。最重要的就是getDelay，这个方法需要返回对象过期前的时间。简单说，队列在某些方法处理前，会调用此方法来判断对象有没有超时。

其次，DelayQueue是一个BlockingQueue，其特化的参数是Delayed。（不了解BlockingQueue的同学，先去了解BlockingQueue再看本文）
Delayed扩展了Comparable接口，比较的基准为延时的时间值，Delayed接口的实现类getDelay的返回值应为固定值（final）。DelayQueue内部是使用PriorityQueue实现的。

总结，DelayQueue的关键元素BlockingQueue、PriorityQueue、Delayed。可以这么说，DelayQueue是一个使用优先队列（PriorityQueue）实现的BlockingQueue，优先队列的比较基准值是时间。本质上即：

**DelayQueue** = **BlockingQueue** + **PriorityQueue**  +  **Delayed**

他们的基本定义如下

```java
public interface Comparable<T> {
    public int compareTo(T o);
}
```

```java
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}
```

```java
public class DelayQueue<E extends Delayed> implements BlockingQueue<E> { 
    private final PriorityQueue<E> q = new PriorityQueue<E>();
}
```



DelayQueue内部的实现使用了一个优先队列。当调用DelayQueue的offer方法时，把Delayed对象加入到**优先队列q**中。如下：

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E first = q.peek();
        q.offer(e);
        if (first == null || e.compareTo(first) < 0)
            available.signalAll();
        return true;
    } finally {
        lock.unlock();
    }
}
```



DelayQueue的take方法，把**优先队列q**的first拿出来（peek），如果没有达到延时阀值，则进行await处理。如下：

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null) {
                available.await();
            } else {
                long delay =  first.getDelay(TimeUnit.NANOSECONDS);
                if (delay > 0) {
                    long tl = available.awaitNanos(delay);
                } else {
                    E x = q.poll();
                    assert x != null;
                    if (q.size() != 0)
                        available.signalAll(); // wake up other takers
                    return x;

                }
            }
        }
    } finally {
        lock.unlock();
    }
}
```







