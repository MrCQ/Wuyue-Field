# Java 多线程大排档

## 进程与线程的区别：
* 进程

它是内存中的一段独立的空间，可以负责当前应用程序的运行。当前这个进程负责调度当前程序中的所有运行细节（操作系统为进程分配一块独立的运行空间

* 线程

它是位于进程中，负责当前进程中的某个具备独立运行资格的空间（进程为线程分配一块独立运行的空间）
进程是负责某个程序的执行，线程是负责进程中某个独立功能的运行，一个进程至少要包含一个线程

* 多线程

在一个进程中可以开启多个线程，让多个线程同时去完成某项任务。使用多线程的目的是为了提高程序的执行效率。


## 线程的基本使用方式

### 继承Thread类

```java

public class TestThread extends Thread {
    public static void main(String[] args) {
        new TestThread().start();
    }
}

```

1. 集成Thread类，实现自定义线程类
2. 实例化线程类对象
3. 通过start方法启动线程

### 实现Runnable接口

```java
public class TestThread implements Runnable{
    public static void main(String[] args) {
        new Thread(new TestThread()).start();
    }

    @Override
    public void run() {
        //...
    }
}
```

1. 实现Runnable接口，完成Run方法中的逻辑代码
2. 创建Thread对象，并将实现Runnable接口的类实例对象作为参数传入
3. 通过Thread的start方法启动线程

### 通过Callable接口实现带返回值

```java
public class TestThread implements Callable<Void>{
    public static void main(String[] args) {
        //两种方式：Callable+Future, Callable+FutureTask

        Callable<Void> callable = new TestThread();
        FutureTask<Void> futureTask = new FutureTask<Void>(callable);

        new Thread(futureTask).start();
        //Future<Void> future = Executors.newCachedThreadPool().submit(callable);

        try {
            futureTask.get(); //拿到返回值
            //future.get()
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public Void call() throws Exception {
        return null;
    }
}
```

## 线程池

线程的创建于销毁是需要占用系统资源的，在系统资源一定的情况之下，所能够创建的线程数是固定的，通常，我们会“缓存”一部分线程，按需申请使用，用完放回线程池

通过Executors的静态方法我们可以获取到四类线程池

* `newCachedThreadPool` 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
* `newFixedThreadPool` 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
* `newScheduledThreadPool` 创建一个定长线程池，支持定时及周期性任务执行。
* `newSingleThreadExecutor` 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。

### 线程池的基本使用

```java
public class TestThreadPool {
    public static void main(String[] args) {
        Executors.newFixedThreadPool(2).execute(new Runnable() {
            @Override
            public void run() {
                //...
            }
        });
        Future<Object> future = Executors.newSingleThreadExecutor().submit(new Callable<Object>() {
            @Override
            public Object call() throws Exception {
                return 1;
            }
        });
        Executors.newCachedThreadPool().execute(new Runnable() {
            @Override
            public void run() {
                //...
            }
        });
        Executors.newScheduledThreadPool(2).scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                //...
            }
        },1000,3000, TimeUnit.SECONDS);
    }
}
```

注意两个方法：`execute`和`submit`，两者区别在于，前者用于Runnable方法，后者用于Callable并返回Future以获取线程返回值

### 线程池的基本原理

Executors 的四个静态方法返回的都是`ExecutorService`类型的线程池对象，`ExecutorService`为定义线程池基本方法的接口，真正的实现类是`ThreadPoolExecutor`因而，`java.uitl.concurrent.ThreadPoolExecutor`类是线程池中最核心的一个类

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}
```

**几个核心参数：**

- `corePoolSize`：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中

- `maximumPoolSize`：线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程，在线程数达到corePoolSize之前，来一个任务创建一个线程，达到corePoolSize后，来的任务会被放到等待队列中，直到队列满，当等待队列满了之后，线程池会尝试为新来的任务创建新的线程进行处理，直到线程总数达到maximumPoolSize，再之后，对于新来的任务就需要根据拒绝策略进行相应的操作。

- `keepAliveTime`：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0；

- `unit`：参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性：

```
TimeUnit.DAYS;               //天
TimeUnit.HOURS;             //小时
TimeUnit.MINUTES;           //分钟
TimeUnit.SECONDS;           //秒
TimeUnit.MILLISECONDS;      //毫秒
TimeUnit.MICROSECONDS;      //微妙
TimeUnit.NANOSECONDS;       //纳秒
```

- `workQueue`：一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：

```
ArrayBlockingQueue; 基于数组的先进先出队列，此队列创建时必须指定大小；
LinkedBlockingQueue; 基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；
SynchronousQueue; 这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。
```

- threadFactory：线程工厂，主要用来创建线程；

- handler：表示当拒绝处理任务时的策略，有以下四种取值：

```
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务 
```

线程提交给线程池进行处理，提交过程有几点细节：

1. 如果当前线程池中的线程数目小于corePoolSize，则每来一个任务，就会创建一个线程去执行这个任务；
2. 如果当前线程池中的线程数目>=corePoolSize，则每来一个任务，会尝试将其添加到任务缓存队列当中，若添加成功，则该任务会等待空闲线程将其取出去执行；**若添加失败（一般来说是任务缓存队列已满），则会尝试创建新的线程去执行这个任务**；
3. 如果当前线程池中的线程数目达到maximumPoolSize，则会采取任务拒绝策略进行处理；
4. 如果线程池中的线程数量大于 corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止，直至线程池中的线程数目不大于corePoolSize；如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过keepAliveTime，线程也会被终止。

### 如何正确设置线程池的大小

一般需要根据任务的类型来配置线程池大小：

* 如果是CPU密集型任务，就需要尽量压榨CPU，参考值可以设为 NCPU+1
* 如果是IO密集型任务，参考值可以设置为2*NCPU

### 阻塞队列

从collection接触到的基本都是非阻塞队列，比如PriorityQueue、LinkedList（LinkedList是双向链表，它实现了Dequeue接口）。

使用非阻塞队列的时候有一个很大问题就是：它不会对当前线程产生阻塞，那么在面对类似消费者-生产者的模型时，就必须额外地实现同步策略以及线程间唤醒策略，这个实现起来就非常麻烦。但是有了阻塞队列就不一样了，它会对当前线程产生阻塞，比如一个线程从一个空的阻塞队列中取元素，此时线程会被阻塞直到阻塞队列中有了元素。当队列中有元素后，被阻塞的线程会自动被唤醒（不需要我们编写代码去唤醒）。这样提供了极大的方便性。

几种最常用的阻塞队列：

* `ArrayBlockingQueue`：基于数组实现的一个阻塞队列，在创建ArrayBlockingQueue对象时必须制定容量大小。并且可以指定公平性与非公平性，默认情况下为非公平的，即不保证等待时间最长的队列最优先能够访问队列。

* `LinkedBlockingQueue`：基于链表实现的一个阻塞队列，在创建LinkedBlockingQueue对象时如果不指定容量大小，则默认大小为Integer.MAX_VALUE。

* `PriorityBlockingQueue`：以上2种队列都是先进先出队列，而PriorityBlockingQueue却不是，它会按照元素的优先级对元素进行排序，按照优先级顺序出队，每次出队的元素都是优先级最高的元素。注意，此阻塞队列为无界阻塞队列，即容量没有上限（通过源码就可以知道，它没有容器满的信号标志），前面2种都是有界队列。

* `DelayQueue`：基于PriorityQueue，一种延时阻塞队列，DelayQueue中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。DelayQueue也是一个无界队列，因此往队列中插入数据的操作（生产者）永远不会被阻塞，而只有获取数据的操作（消费者）才会被阻塞。

几个基本方法：

```java
　　put(E e)	用来向队尾存入元素，如果队列满，则等待；

　　take()		方法用来从队首取元素，如果队列为空，则等待；

　　offer(E e,long timeout, TimeUnit unit)用来向队尾存入元素，如果队列满，则等待一定的时间，当时间期限达到时，如果还没有插入成功，则返回false；否则返回true；

　　poll(long timeout, TimeUnit unit)	从队首取元素，如果队列空，则等待一定的时间，当时间期限达到时，如果取到，则返回null；否则返回取得的元素；

```

**实例案例：**

最经典的生产者消费者问题就完成可以通过阻塞队列来完成，非常直接简单

**基本原理：**

阻塞队列实现的关键依赖于以下几个变量：

```java
/** Main lock guarding all access */
private final ReentrantLock lock;
/** Condition for waiting takes */
private final Condition notEmpty;
/** Condition for waiting puts */
private final Condition notFull;
```

详见：Lock 的条件变量 Condition

### CompletionService 自动获得最先完成任务

在多线程中，可以通过 Callable 实现带返回值的线程处理，通过 Future 对象的 get 方法拿到最终的返回值，但是 future 对象的 get 方法是个阻塞方法，需要一直等待到得到返回值

当提交多个 Callable 任务，得到多个处理返回值得时候，需要利用容器保存多个 Future 存根，同时，为了拿到最终返回值，需要通过遍历的方式获取，但是哪一个 Future 先得到返回值这是不确定的，在遍历过程中，可能一直阻塞在前面 Future 对象的 get 方法上，导致后面实际已经得到返回值的 Future 一直得不到处理。

所以，如何能够自动将已经获取到返回值的 Future 能够优先得到？ 答案就是 `CompletionService`

```java
public class TestThreadPool {
    public static void main(String[] args) throws Exception{
        CompletionService<String> completionService = new ExecutorCompletionService<String>(Executors.newSingleThreadExecutor());

        for(int i = 0; i < 10; i++) {
            completionService.submit(new Callable<String>() {
                @Override
                public String call() throws Exception {
                    //...time-costing process
                    return "...";
                }
            });
        }

        for(int i = 10; i < 10; i++) {
            Future<String> future = completionService.take();
            future.get();
        }
        
    }
}
```

使用 CompletionService 来维护处理线程的返回结果时，**主线程总是能够拿到最先完成的任务的返回值**，而不管它们加入线程池的顺序。


## CompletableFuture 基本原理与应用

为什么要引入 CompletableFuture，而不是直接使用 Future 呢？

* Future.get() 方法会阻塞线程，一直阻塞，直到其所对应的任务完成或因为异常退出
* Future.get(long, TimeUnit) 可以一定的时间内超时退出，而不会像前一个方法那样一直阻塞线程。但是这对系统响应性的改进是治标不治本。

Java 8中引入CompletableFuture，可以通过`CompletableFuture thenAccept(Consumer<? super T> action)` 实现异步操作。


## ThreadLocal 基本使用与原理

`ThreadLocal` 即线程本地变量， `ThreadLocal`为变量在每个线程中都创建了一个副本，每个线程访问自己内部的副本变量，这样就不存在线程之间的资源竞争问题

`ThreadLocal`提供的几个基本方法：

```java
public T get() { }
public void set(T value) { }
public void remove() { }
protected T initialValue() { }
```

基本使用示例：

```java
public class Test {
    public static void main(String[] args) {
        MyRunnable sharedRunnableInstance = new MyRunnable();
        Thread thread1 = new Thread(sharedRunnableInstance);
        Thread thread2 = new Thread(sharedRunnableInstance);
        thread1.start();
        thread2.start();

    }

    static class MyRunnable implements Runnable {

        private static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>(){
            @Override
            protected Integer initialValue() {
                return 10;
            }
        };

        @Override
        public void run() {
            threadLocal.set((int) (Math.random() * 100D));

            System.out.println("before : " + threadLocal.get());
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {

            }
            System.out.println("after : " + threadLocal.get());
        }
    }

}
```

从以上示例可以看出，即便`ThreadLocal`对象是静态对象，正常静态对象应该是归所有实例对象共有（相互影响），但是定义为`ThreadLocal`之后，为每个线程创建自己的本地变量.


## 同步，并发控制与锁机制

谈到多线程控制，通常提及的属并发控制比较多，同步控制也相对比较常见

### 线程同步

同步控制最简单的： `wait 与 notify` 的组合

java 中**任何对象都能被当做是锁**，即便是 Class 也能够通过其所对应的 class 对象而被当做锁，只要是锁，就包含以下三个方法：

* wait()
* notify()
* notifyAll()

要执行以上三个方法，必须先锁定该对象，因为对应的状态变量也是由该对象锁保护的

```java
 public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();
        synchronized (obj) {
            obj.wait();
        }
    }


 public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();
        synchronized (obj) {
            obj.notify(); // 唤醒一个
			  //obj.notifyAll(); 唤醒所有
        }
    }
```

在执行锁对象 obj 的 wait 方法前，必须先利用`synchronized`锁操作锁住该对象，其中**执行wait操作后，会自动释放锁**，并休眠等待，知道其他线程调用了 obj 的 `notify`方法，该线程才会苏醒重新获取锁，完成后面的执行任务

案例：利用 `wait & notify` 实现生产者消费者经典问题

```java
/**
 * 单生产者单消费者,
 * 　用wait notify实现线程间通信
 */
public class SimpleProducerAndConsumer {
	  private static final int MAX = 10;
    private static final Object lock = new Object();
    public static void main(String[] args) {
        new ProducerThread().start();
        new ConsumerThread().start();
    }

    private static class ProducerThread extends Thread{
        @Override
        public void run() {
            super.run();
            while (true){
                synchronized (lock){
                    if (Res.list.size() == MAX){

                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    Res.list.add("h");
                    System.out.println(getName() + ": 生产者生产一个元素");
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    lock.notify();
                }

            }


        }
    }

    private static class ConsumerThread extends Thread{
        @Override
        public void run() {
            super.run();
            while(true){
                synchronized (lock){
                    while(Res.list.size() == 0){

                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    Res.list.remove(0);
                    System.out.println(getName() + ": 消费者消费一个元素");
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    lock.notify();
                }

            }

        }
    }


    private static class Res{
        static ArrayList<String> list = new ArrayList<>();
    }
}
```

### 并发控制与锁

锁是并发控制以及同步控制中最常见的机制，最常用的由两种方式：

* synchronized
* Lock

主要区别在于:

* syncrhonize 重量级，悲观锁实现方式，由jvm完成加锁个解锁操作，语义上很清晰，可以进行很多优化，有适应自旋，锁消除，锁粗化，轻量级锁，偏向锁等等
* Lock 更加灵活，采用乐观锁实现方式，且增加了公平锁和锁中断，需要自己从代码层面手动释放

由于 `Synchronized` 采用的是悲观锁的实现方式，当出现竞争时，需要进行锁的休眠和唤醒，而这本身是一件比较耗费资源的事情。所以在资源竞争不是很激烈的情况下，`Synchronized`的性能要优于`ReetrantLock`，但是在资源竞争很激烈的情况下，`Synchronized`的性能会下降几十倍，但是`ReetrantLock`的性能能维持常态

#### Synchronized

Synchronized可用于一下几处场景：

* 同步代码块，锁对象是synchronized括号中对应配置的对象，如

```java
public void test() {
	Object obj = new Object();
	synchronized(obj) {
		//...
	}
}
```
* 普通成员方法，锁对象是当前实例对象

```java
public synchronized void test() {
	//...
}
```

* 类静态方法，所对象是当前类的class对象

```java
public static synchronized void test() {
	//...
}
```

这个比较特殊，因为静态方法对于当前类的所有对象都是可用的，所以为静态方法加锁会作用到类以及类所实例化的全部对象。


#### Lock

Lock只是定义的锁接口，一般使用的实现类是:

* `ReentrantLock` 可重入锁，即已获得锁的线程在此执行lock时候，匹配验证为锁拥有者，不会阻塞自身
* `ReadWriteLock` 读写锁，读锁和写锁分离，读锁不存在竞争，适合读多写少的场景
* `ReentrantReadWriteLock` 可重入读写锁

##### Lock的一般使用方式

Lock 接口下锁包含的基本方法有：

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

* `lock()`方法是平常使用得最多的一个方法，就是用来获取锁。如果锁已被其他线程获取，则进行等待

```java
Lock lock = new ReentrantLock();
lock.lock();
try{
    //...
}catch(Exception ex){
   //...  
}finally{
    lock.unlock();   //手动释放锁
}
```

* `tryLock()`方法是有返回值的，它表示用来尝试获取锁，如果获取成功，则返回 true，如果获取失败（即锁已被其他线程获取），则返回 false，也就说这个方法无论如何都会立即返回。在拿不到锁时不会一直在那等待。

```java
Lock lock = new ReentrantLock();
if(lock.tryLock()){
	try{
    	//...
	}catch(Exception ex){
     //...
	}finally{
    lock.unlock();   //手动释放锁
	}
}
```

* `tryLock(long time, TimeUnit unit)`方法和`tryLock()`方法是类似的，只不过区别在于这个方法在拿不到锁时会等待一定的时间，在时间期限之内如果还拿不到锁，就返回false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。


##### Lock 的条件变量 Condition

条件变量Condition很大一个程度上是为了解决Object.wait/notify/notifyAll难以使用的问题。完成线程之间的通信问题。条件（也称为条件队列 或条件变量）为线程提供了一个含义，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”）。等待提供一个条件的主要属性是：以原子方式 释放相关的锁，并挂起当前线程，**就像 Object.wait 做的那样**

**在Condition中，用await()替换wait()，用signal()替换notify()，用signalAll()替换notifyAll()，传统线程的通信方式，Condition都可以实现，这里注意，Condition是被绑定到Lock上的，要创建一个Lock的Condition必须用newCondition()方法。**

Condition和传统的线程通信没什么区别，**Condition的强大之处在于它可以为多个线程间建立不同的Condition**

通过一个使用案例来说明，其实就是阻塞队列的简单实现方式:

```java
class BoundedBuffer {  
   final Lock lock = new ReentrantLock();//锁对象  
   final Condition notFull  = lock.newCondition();//写线程条件   
   final Condition notEmpty = lock.newCondition();//读线程条件   
  
   final Object[] items = new Object[100];//缓存队列  
   int putptr/*写索引*/, takeptr/*读索引*/, count/*队列中存在的数据个数*/;  
  
   public void put(Object x) throws InterruptedException {  
     lock.lock();  
     try {  
       while (count == items.length)//如果队列满了   
         notFull.await();//阻塞写线程  
       items[putptr] = x;//赋值   
       if (++putptr == items.length) putptr = 0;//如果写索引写到队列的最后一个位置了，那么置为0  
       ++count;//个数++  
       notEmpty.signal();//唤醒读线程  
     } finally {  
       lock.unlock();  
     }  
   }  
  
   public Object take() throws InterruptedException {  
     lock.lock();  
     try {  
       while (count == 0)//如果队列为空  
         notEmpty.await();//阻塞读线程  
       Object x = items[takeptr];//取值   
       if (++takeptr == items.length) takeptr = 0;//如果读索引读到队列的最后一个位置了，那么置为0  
       --count;//个数--  
       notFull.signal();//唤醒写线程  
       return x;  
     } finally {  
       lock.unlock();  
     }  
   }   
 }  
```

##### 读写锁

ReadWriteLock 接口下得基本方法：

```java
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading.
     */
    Lock readLock();
 
    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing.
     */
    Lock writeLock();
}
```

读写锁的几个基本特点：

1. 读锁相互之间不影响（即不存在竞争关系），读写锁之间有影响
2. 当有写锁时候，读锁申请需等待，没有写锁时候直接获得读锁
3. 当写锁申请时候，如果没有读锁直接获得，若有读锁，需等待读操作完成，并且因为有写锁等待，后来的读锁申请也需要等待直到写操作完成

实际使用案例： 

```java
public class Test {
    private ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
     
    public static void main(String[] args)  {
        final Test test = new Test();
         
        new Thread(){
            public void run() {
                test.get(Thread.currentThread());
            };
        }.start();
         
        new Thread(){
            public void run() {
                test.get(Thread.currentThread());
            };
        }.start();
         
    }  
     
    public void get(Thread thread) {
        rwl.readLock().lock();
        try {
            long start = System.currentTimeMillis();
             
            while(System.currentTimeMillis() - start <= 1) {
                System.out.println(thread.getName()+"正在进行读操作");
            }
            System.out.println(thread.getName()+"读操作完毕");
        } finally {
            rwl.readLock().unlock();
        }
    }
}
```

#### Lock和synchronized的选择

总结来说，`Lock`和`synchronized`有以下几点不同：

* Lock是一个接口，而synchronized是Java中的关键字，synchronized是内置的语言实现；
* synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；
* Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断；
* 通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。
* Lock可以提高多个线程进行读操作的效率。


参考：

[Java并发编程：线程池的使用 - 海 子 - 博客园](https://www.cnblogs.com/dolphin0520/p/3932921.html)

[Java并发编程：Lock - 海 子 - 博客园](https://www.cnblogs.com/dolphin0520/p/3923167.html)

[Java并发编程：阻塞队列 - 海 子 - 博客园](http://www.cnblogs.com/dolphin0520/p/3932906.html)

[Java ThreadLocal的使用 | 并发编程网 – ifeve.com](http://ifeve.com/java-threadlocal%E7%9A%84%E4%BD%BF%E7%94%A8/)