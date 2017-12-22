# The Art of Multi-Thread 7 —— 闭锁、同步屏障、信号量详解


## 闭锁：CountDownLatch

### 使用场景

若有多条线程，其中一条线程需要等到其他**所有**线程准备完所需的资源后才能运行，这样的情况可以使用闭锁。

## 代码实现

```java
// 初始化闭锁，并设置资源个数
CountDownLatch latch = new CountDownLatch(2);

Thread t1 = new Thread( new Runnable(){
    public void run(){
        // 加载资源1
        // 加载资源的代码……
        // 本资源加载完后，闭锁-1
        latch.countDown();
    }
} ).start();

Thread t2 = new Thread( new Runnable(){
    public void run(){
        // 加载资源2
        // 资源加载代码……
        // 本资源加载完后，闭锁-1
        latch.countDown();
    }
} ).start();

Thread t3 = new Thread( new Runnable(){
    public void run(){
        // 本线程必须等待所有资源加载完后才能执行
        latch.await();
        // 当闭锁数量为0时，await返回，执行接下来的任务
        任务代码……
    }
} ).start();
```





******





# 同步屏障：CyclicBarrier

### 使用场景

若有多条线程，他们到达屏障时将会被阻塞，只有当所有线程都到达屏障时才能打开屏障，所有线程同时执行，若有这样的需求可以使用同步屏障。此外，当屏障打开的同时还能指定执行的任务。





### 闭锁 与 同步屏障 的区别

- **闭锁只会阻塞一条线程**，目的是**为了让该条任务线程满足条件后执行**；
- 而同步屏障会**阻塞所有线程**，目的是为了**让所有线程同时执行**（实际上并不会同时执行，而是尽量把线程启动的时间间隔降为最少）。



### 代码实现

```java
// 创建同步屏障对象，并制定需要等待的线程个数 和 打开屏障时需要执行的任务
CyclicBarrier barrier = new CyclicBarrier(3,new Runnable(){
    public void run(){
        //当所有线程准备完毕后触发此任务
    }
});

// 启动三条线程
for( int i=0; i<3; i++ ){
    new Thread( new Runnable(){
        public void run(){
            // 等待，（每执行一次barrier.await，同步屏障数量-1，直到为0时，打开屏障）
            barrier.await();
            // 任务
            任务代码……
        }
    } ).start();
}
```





******





# 信号量：Semaphore



### 使用场景

若有m个资源，但有n条线程（n>m），因此同一时刻只能允许m条线程访问资源，此时可以使用Semaphore控制访问该资源的线程数量。





### 代码实现

```java
// 创建信号量对象，并给予3个资源
Semaphore semaphore = new Semaphore(3);

// 开启10条线程
for ( int i=0; i<10; i++ ) {
    new Thread( new Runnbale(){
        public void run(){
            // 获取资源，若此时资源被用光，则阻塞，直到有线程归还资源
            semaphore.acquire();
            // 任务代码
            ……
            // 释放资源
            semaphore.release();
        }
    } ).start();
}
```































