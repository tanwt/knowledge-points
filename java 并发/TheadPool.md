### . ThreadPool介绍

&emsp; Java 中的线程池是运用场景最多的并发框架，几乎所有需要异步或并发执行任务的程序都可以使用线程池。在开发过程中，合理使用线程池有如下好处：
1. 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。


### . 线程池的继承体系

&emsp;我们从线程池的源码来了解线程池的继承体系

```
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

&emsp; 从上面给出的ThreadPoolExecutor类的代码可以知道，ThreadPoolExecutor继承了AbstractExecutorService，我们来看一下AbstractExecutorService的实现：

```
public abstract class AbstractExecutorService implements ExecutorService {
 
     
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) { };
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) { };
    public Future<?> submit(Runnable task) {};
    public <T> Future<T> submit(Runnable task, T result) { };
    public <T> Future<T> submit(Callable<T> task) { };
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                            boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
    };
}
```
　
&emsp;AbstractExecutorService是一个抽象类，它实现了ExecutorService接口。

```
public interface ExecutorService extends Executor {
 
    void shutdown();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
 
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
而ExecutorService又是继承了Executor接口，我们看一下Executor接口的实现：
```
public interface Executor {
    void execute(Runnable command);
}
```
到这里，大家应该明白了ThreadPoolExecutor、AbstractExecutorService、ExecutorService和Executor几个之间的关系了。

　　Executor是一个顶层接口，在它里面只声明了一个方法execute(Runnable)，返回值为void，参数为Runnable类型，从字面意思可以理解，就是用来执行传进去的任务的；

　　然后ExecutorService接口继承了Executor接口，并声明了一些方法：submit、invokeAll、invokeAny以及shutDown等；

　　抽象类AbstractExecutorService实现了ExecutorService接口，基本实现了ExecutorService中声明的所有方法；

　　然后ThreadPoolExecutor继承了类AbstractExecutorService。


### . 线程池的四个构造函数,四个参数详细解释原理

&emsp; ThreadPoolExecutor提供了四个构造函数:

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue)

public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory)

public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler)

public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)

```
下面解释下一下构造器中各个参数的含义：
- **corePoolSize（线程池的基本大小）**

&emsp;核心线程大小,当提交一个任务到线程池时，线程池会创建一个核心线程来执行任务，即使其他空闲的核心线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池核心线程大小时就不再创建。如果调用线程池的prestartAllCoreThreads()或者prestartCoreThread()方法，线程池会提前创建线程。

- **maximumPoolSize**

&emsp;线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程。


```
ps:
有很多小伙伴无法理解corePoolSize 和maximumPoolSize 的意思。用个简单的例子来讲：
假设TheadPool 是一个工厂，负责加工某个东西。corePoolSize 核心线程的意思
就相当于是这个工厂的长期固定工人，负责加工东西。当一个东西进入工厂，一个
工人就负责加工。当所有工人都在工作时，再添加一个东西，就会被放在一边等待
被加工。
但是某天工厂需要加工的东西太多了，工厂都放不下了，就需要再招一些临时工。
maximumPoolSize 最大线程数就是说corePoolSize 核心工人（核心线程）之外
再招的临时工（临时线程）不能超过这个数，超过这个数，工厂就会亏钱，抛出异常，无法运转。
```

- **keepAliveTime**

&emsp;表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0。

- **unit**

&emsp;keepAliveTime的单位，TimeUnit是一个枚举类型，其包括：
```
NANOSECONDS ： 1微毫秒 = 1微秒 / 1000
MICROSECONDS ： 1微秒 = 1毫秒 / 1000
MILLISECONDS ： 1毫秒 = 1秒 /1000
SECONDS ： 秒
MINUTES ： 分
HOURS ： 小时
DAYS ： 天
```

- **workQueue**

&emsp;该线程池中的任务队列：维护着等待执行的Runnable对象
当所有的核心线程都在干活时，新添加的任务会被添加到这个队列中等待处理，如果队列满了，则新建非核心线程执行任务。

常用的workQueue类型：

1. ArrayBlockingQueue：是一个基于数据结构的有界阻塞队列，此队列按FIFO（先进先出）原则对元素进行排序。
2. LinkedBlockingQueue：一个基于链表数据结构的阻塞队列，此队列按FIFO 排序元素，吞吐量要高于ArrayBlockingQueue。由于这个队列没有最大值限制，即所有超过核心线程数的任务都将被添加到队列中，这也就导致了maximumPoolSize的设定失效，因为总线程数永远不会超过corePoolSize。静态工厂Executors.newFixedThreadPool() 使用了这个队列。
3. SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态。吞吐量通常高于LinkedBlockingQueue ，静态工厂方法Executors.newCachedThreadPool() 使用了这个队列。
4. PriorityBlockingQueue：一个具有优先级的队列。


- **threadFactory**

&emsp;线程工厂，主要用来创建线程。



- **handler**

&emsp;当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认是AbortPolicy ，表示无法处理新任务是抛出异常。

表示当拒绝处理任务时的策略，有以下四种取值。

```
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）。
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```






### . 原理

&emsp;提交一个任务到线程池中，线程池的处理流程：
1. 判断线程池里的核心线程是否都在执行任务，如果不是（核心线程空闲或者还有核心线程没有被创建）则创建一个新的工作线程来执行任务。如果核心线程都在执行任务，则进入下个流程。
2. 线程池判断工作队列是否已满，如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。
3. 判断线程池里的线程是否都处于工作状态，如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务。


![image](C:\Users\Administrator\Desktop\图片\笔记\面试题\TIM截图20180814150809.png)

&emsp;ThreadPoolExecutor 执行execute() 方法分下面4种情况：
1. 如果当前运行的线程小于corePoolSize ，则创建新线程来执行任务（注意：执行这一步需要获取全局锁）。
2. 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。
3. 如果无法将任务加入BlockingQueue（队列已满），则创建新的非核心线程来处理任务（注意：执行这一步需要获取全局锁）。
4. 如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution() 方法。

&emsp;上面的流程让我们很直观的了解到了线程池的工作流程。下面我们通过源码来进行解读：
1. ThreadPoolExecutor的execute()方法

```
 public void execute(Runnable command) {
          if (command == null)
              throw new NullPointerException();
　　　　　　 //如果线程数大于等于基本线程数或者线程创建失败，将任务加入队列
          if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
　　　　　　　　　　//线程池处于运行状态并且加入队列成功
              if (runState == RUNNING && workQueue.offer(command)) {
                  if (runState != RUNNING || poolSize == 0)
                      ensureQueuedTaskHandled(command);
              }
　　　　　　　　　//线程池不处于运行状态或者加入队列失败，则创建线程（创建的是非核心线程）
             else if (!addIfUnderMaximumPoolSize(command))
　　　　　　　　　　　//创建线程失败，则采取阻塞处理的方式
                 reject(command); // is shutdown or saturated
         }
}
```
2. 创建线程的方法：addIfUnderCorePoolSize(command)
```
private boolean addIfUnderCorePoolSize(Runnable firstTask) {
         Thread t = null;
         final ReentrantLock mainLock = this.mainLock;
         mainLock.lock();
         try {
             if (poolSize < corePoolSize && runState == RUNNING)
                 t = addThread(firstTask);
         } finally {
             mainLock.unlock();
         }
         if (t == null)
             return false;
         t.start();
         return true;
     }
```
addThread()
```
private Thread addThread(Runnable firstTask) {
        Worker w = new Worker(firstTask);
        Thread t = threadFactory.newThread(w);
        if (t != null) {
            w.thread = t;
            workers.add(w);
            int nt = ++poolSize;
            if (nt > largestPoolSize)
                largestPoolSize = nt;
        }
        return t;
    }
```
这里将线程封装成工作线程worker，并放入工作线程组里，worker类的方法run方法：

```
public void run() {
            try {
                Runnable task = firstTask;
                firstTask = null;
                while (task != null || (task = getTask()) != null) {
                    runTask(task);
                    task = null;
                }
            } finally {
                workerDone(this);
            }
        }
```
线程池中的线程执行任务分两种情况：
1. 在execute() 方法中创建一个线程时，会让这个线程执行当前任务。
2. 这个现场执行完任务后，会反复从BlockingQueue 获取任务来执行。