# The Art of Multi-Thread 9 批量获取多条线程的执行结果

# 方法一：自己维护返回结果
```java
// 创建一个线程池
ExecutorService executorService = Executors.newFixedThreadPool(10);

// 存储执行结果的List
List<Future<String>> results = new ArrayList<Future<String>>();

// 提交10个任务
for ( int i=0; i<10; i++ ) {
    Future<String> result = executorService.submit( new Callable<String>(){
        public String call(){
            int sleepTime = new Random().nextInt(1000);
            Thread.sleep(sleepTime);
            return "线程"+i+"睡了"+sleepTime+"秒";
        }
    } );
    // 将执行结果存入results中
    results.add( result );
}

// 获取10个任务的返回结果
for ( int i=0; i<10; i++ ) {
    // 获取包含返回结果的future对象
    Future<String> future = results.get(i);
    // 从future中取出执行结果（若尚未返回结果，则get方法被阻塞，直到结果被返回为止）
    String result = future.get();
    System.out.println(result);
}
```
此方法的弊端：
1. 需要自己创建容器维护所有的返回结果，比较麻烦；
2. 从list中遍历的每个Future对象并不一定处于完成状态，这时调用**get()方法就会被阻塞住**，如果系统是设计成每个线程完成后就能根据其结果继续做后面的事，这样对于处于list后面的但是先完成的线程就会增加了额外的等待时间。



# 方法二：使用ExecutorService的invokeAll函数
本方法能解决第一个弊端，即并不需要自己去维护一个存储返回结果的容器。当我们需要获取线程池所有的返回结果时，只需调用invokeAll函数即可。 
但是，这种方式需要你自己去维护一个用于存储任务的容器。
```java
// 创建一个线程池
ExecutorService executorService = Executors.newFixedThreadPool(10);

// 创建存储任务的容器
List<Callable<String>> tasks = new ArrayList<Callable<String>>();

// 提交10个任务
for ( int i=0; i<10; i++ ) {
    Callable<String> task = new Callable<String>(){
        public String call(){
            int sleepTime = new Random().nextInt(1000);
            Thread.sleep(sleepTime);
            return "线程"+i+"睡了"+sleepTime+"秒";
        }
    };
    executorService.submit( task );
    // 将task添加进任务队列
    tasks.add( task );
}

// 获取10个任务的返回结果
List<Future<String>> results = executorService.invokeAll( tasks );

// 输出结果
for ( int i=0; i<10; i++ ) {
    // 获取包含返回结果的future对象
    Future<String> future = results.get(i);
    // 从future中取出执行结果（若尚未返回结果，则get方法被阻塞，直到结果被返回为止）
    String result = future.get();
    System.out.println(result);
}
```



# 方法三：使用CompletionService
CompletionService内部维护了一个阻塞队列，只有执行完成的任务结果才会被放入该队列，这样就确保执行时间较短的任务率先被存入阻塞队列中。
```java
ExecutorService exec = Executors.newFixedThreadPool(10);

final BlockingQueue<Future<Integer>> queue = new LinkedBlockingDeque<Future<Integer>>(  
                10);  
        //实例化CompletionService  
        final CompletionService<Integer> completionService = new ExecutorCompletionService<Integer>(  
                exec, queue); 

// 提交10个任务
for ( int i=0; i<10; i++ ) {
    executorService.submit( new Callable<String>(){
        public String call(){
            int sleepTime = new Random().nextInt(1000);
            Thread.sleep(sleepTime);
            return "线程"+i+"睡了"+sleepTime+"秒";
        }
    } );
}

// 输出结果
for ( int i=0; i<10; i++ ) {
    /* 获取包含返回结果的future对象（若整个阻塞队列中还没有一条线程返回结果，那么调用take将会被阻塞，
     * 当然你可以调用poll，不会被阻塞，若没有结果会返回null，poll和take返回正确的结果后会将该结果从队列中删除）
     */
    Future<String> future = completionService.take();
    // 从future中取出执行结果，这里存储的future已经拥有执行结果，get不会被阻塞
    String result = future.get();
    System.out.println(result);
}
```








