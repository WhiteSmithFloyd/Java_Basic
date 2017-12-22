# The Art of Multi-Thread 6 —— 进程间通讯与同步

## 共享变量

例子

```java
// 用于控制线程当前的执行状态
private volatile boolean running = false;

// 开启一条线程
Thread thread = new Thread(new Runnable(){
    void run(){
        // 开关
        while(!running){
            Thread.sleep(1000);
        }
        // 执行线程任务
        doSometing();
    }
}).start();

// 开始执行
public void start(){
    running = true;
}
```





### 等待/通知机制

等待/通知机制的实现由Java完成，我们只需调用Object类的几个方法即可。

- wait()：将当前线程的状态改为“等待态”，加入**等待队列**，**释放锁**；直到当前线程发生中断或调用了notify方法，**这条线程才会被从等待队列转移到同步队列，此时可以开始竞争锁**。
- wait(long)：和wait()功能一样，只不过多了个超时动作。一旦超时，就会继续执行wait之后的代码，它不会抛超时异常！
- notify()：将等待队列中的一条线程转移到同步队列中去。
- notifyAll()：将等待队列中的所有线程都转移到同步队列中去。

### 注意点

- 以上方法都必须放在同步块中；
- 并且以上方法都只能由所处同步块的锁对象调用；
- 锁对象A.notify()/notifyAll()只能唤醒由锁对象A wait的线程；
- 调用**notify/notifyAll函数后仅仅是将线程从等待队列转移到阻塞队列**，只有当该线程竞争到锁后，才能从wait方法中返回，继续执行接下来的代码；





### 关于wait方法

wait是指在一个已经进入了同步锁的线程内，**让自己暂时让出同步锁**，以便其他正在等待此锁的线程可以得到同步锁并运行，只有其他线程调用了notify方法（**notify并不释放锁**，**只是告诉调用过wait方法的线程可以去参与获得锁的竞争了**，但不是马上得到锁，因为锁还在别人手里，别人还没释放），**调用wait方法的一个或多个线程就会解除wait状态，重新参与竞争对象锁**，程序如果可以再次得到锁，就可以继续向下运行。

1. wait()、notify()和notifyAll()方法是本地方法，并且为final方法，无法被重写。
2. 当前线程必须拥有此对象的monitor（即锁），才能调用某个对象的wait()方法能让当前线程阻塞，（**这种阻塞是通过提前释放synchronized锁，重新去请求锁导致的阻塞**，这种请求必须有其他线程通过notify()或者notifyAll（）唤醒重新竞争获得锁）
3. 调用某个对象的notify()方法能够唤醒一个正在等待这个对象的monitor的线程，如果有多个线程都在等待这个对象的monitor，则只能唤醒其中一个线程； （**notify()或者notifyAll()方法并不是真正释放锁，必须等到synchronized方法或者语法块执行完才真正释放锁**）


4. 调用notifyAll()方法能够唤醒所有正在等待这个对象的monitor的线程，唤醒的线程获得锁的概率是随机的，取决于cpu调度

#### 例子1

（错误使用导致线程阻塞）：三个线程，线程3先拥有sum对象的锁，然后通过sum.notify()方法通知等待sum锁的线程去获得锁，但是这个时候线程1,2并没有处于wait()导致的阻塞状态，而是在synchronized方法块处阻塞了，所以，这次notify()根本没有通知到线程1,2。然后线程3正常结束，释放掉sum锁，这个时候，线程1就立刻获得了sum对象的锁（通过synchronized获得），然后调用sum.wait()方法释放掉sum的锁，线程2随后获得了sum对象的线程锁（通过synchronized获得），这个时候线程1,2都处于阻塞状态，但是悲催的是，这之后再也没有线程主动调用sum.notify()或者notifyAll()方法显示唤醒这两个线程，所以程序阻塞

```java
public class CyclicBarrierTest {  
      
    public static void main(String[] args) throws Exception {  
        final Sum sum=new Sum();  
          
        new Thread(new Runnable() {  
            @Override  
            public void  run() {  
                try {  
                    synchronized (sum) {  
                        System.out.println("thread3 get lock");  
                        sum.sum();  
                        sum.notifyAll(); //此时唤醒没有作用，没有线程等待  
                        Thread.sleep(2000);  
                        System.out.println("thread3 really release lock");  
                    }  
                      
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
            }  
        }).start();  
          
        new Thread(new Runnable() {  
            @Override  
            public void  run() {  
                try {  
                    synchronized (sum) {  
                        System.out.println("thread1 get lock");  
                        sum.wait();//主动释放掉sum对象锁  
                        System.out.println(sum.total);  
                        System.out.println("thread1 release lock");  
                    }  
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
            }  
        }).start();  
          
        new Thread(new Runnable() {  
            @Override  
            public void  run() {  
                try {  
                    synchronized (sum) {  
                        System.out.println("thread2 get lock");  
                        sum.wait();  //释放sum的对象锁，等待其他对象唤醒（其他对象释放sum锁）  
                        System.out.println(sum.total);  
                        System.out.println("thread2 release lock");  
                    }  
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
            }  
        }).start();  
    }  
            
}  
  
class Sum{  
    public Integer total=0;  
      
    public void  sum() throws Exception{  
        total=100;  
        Thread.sleep(5000);  
    }  
}  
```

运行结果：

```java
thread3 get lock  
thread3 really release lock  
thread2 get lock  
thread1 get lock  
//程序后面一直阻塞  
```





#### 例子2

还是上面程序，顺序不同，把线程3放到最下面。最后线程1,2都因为没有再次获得线程导致线程阻塞

 

运行过程：

线程1先运行获得sum对象锁（通过synchronized），但是随后执行了sum.wait()方法，主动释放掉了sum对象锁，然后线程2获得了sum对象锁（通过synchronized）,也通过sum.wait()失去sum的对象锁，最后线程3获得了sum对象锁（通过synchronized），主动通过sum.notify()通知了线程1或者2，假设是1，线程1重新通过notify()/notifyAll()的方式获得了锁，然后执行完毕，随后线程释放锁，然后这个时候线程2成功获得锁，执行完毕。

```java
public class CyclicBarrierTest {  
      
    public static void main(String[] args) throws Exception {  
        final Sum sum=new Sum();  
          
      
          
        new Thread(new Runnable() {  
            @Override  
            public void  run() {  
                try {  
                    synchronized (sum) {  
                        System.out.println("thread1 get lock");  
                        sum.wait();//主动释放sum对象锁，等待唤醒  
                        System.out.println(sum.total);  
                        System.out.println("thread1 release lock");  
                    }  
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
            }  
        }).start();  
          
        new Thread(new Runnable() {  
            @Override  
            public void  run() {  
                try {  
                    synchronized (sum) {  
                        System.out.println("thread2 get lock");  
                        sum.wait();  //主动释放sum对象锁，等待唤醒  
                        System.out.println(sum.total);  
                        System.out.println("thread2 release lock");  
                    }  
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
            }  
        }).start();  
          
        new Thread(new Runnable() {  
            @Override  
            public void  run() {  
                try {  
                    synchronized (sum) {  
                        System.out.println("thread3 get lock");  
                        sum.sum();  
                        sum.notifyAll();//唤醒其他等待线程（线程1,2）  
                        Thread.sleep(2000);  
                        System.out.println("thread3 really release lock");  
                    }  
                      
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
            }  
        }).start();  
          
          
    }  
            
}  
  
class Sum{  
    public Integer total=0;  
      
    public void  sum() throws Exception{  
        total=100;  
        Thread.sleep(5000);  
    }  
      
}  
```

执行结果

```java
thread1 get lock  
thread2 get lock  
thread3 get lock  
thread3 really release lock  
100  
thread2 release lock  
100  
thread1 release lock  
```







# 管道流

管道流用于在两个线程之间进行字节流或字符流的传递。



### 特点

- 管道流的实现依靠PipedOutputStream、PipedInputStream、PipedWriter、PipedReader。分别对应字节流和字符流。
- 他们与IO流的区别是：IO流是在硬盘、内存、Socket之间流动，而管道流仅在内存中的两条线程间流动。



## 实现

步骤如下： 

1. 在一条线程中分别创建输入流和输出流； 
2. 将输入流和输出流连接起来； 
3. 将输入流和输出流分别传递给两条线程； 
4. 调用read和write方法就可以实现线程间通信。

```java
// 创建输入流与输出流对象
PipedWriter out = new PipedWriter();
PipedReader in = new PipedReader();

// 连接输入输出流
out.connect(in);

// 创建写线程
class WriteThread extends Thread{
    private PipedWriter out;

    public WriteThread(PipedWriter out){
        this.out = out;
    }

    public void run(){
        out.write("hello concurrent world!");
    }
}

// 创建读线程
class ReaderThread extends Thread{
    private PipedReader in;

    public ReaderThread(PipedReader in){
        this.in = in;
    }

    public void run(){
        in.read();
    }
}

```



# join

### 作用

- join能将并发执行的多条线程串行执行；
- join函数属于Thread类，通过一个thread对象调用。当在线程B中执行threadA.join()时，线程B将会被阻塞(**底层调用wait方法**)，等到threadA线程运行结束后才会返回join方法。
- 被等待的那条线程可能会执行很长时间，因此join函数会抛出InterruptedException。当调用threadA.interrupt()后，join函数就会抛出该异常。

```java
public static void main(String[] args){

    // 开启一条线程
    Thread t = new Thread(new Runnable(){
        public void run(){
            // doSometing
        }
    }).start();

    // 调用join，等待t线程执行完毕
    try{
        t.join();
    }catch(InterruptedException e){
        // 中断处理……
    }

}
```































































































































