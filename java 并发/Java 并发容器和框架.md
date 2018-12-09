### . ConcurrentHashMap就一定是线程安全的吗?
答：

&emsp; 虽然ConcurrentHashMap是线程安全的，但是如果你操作的是其本身，并如果使用不当，也会造成很多线程安全问题。 

以下代码来源：https://blog.csdn.net/u011983531/article/details/65936945

```
public class ConcurrentHashMapTest {

    private static final ConcurrentMap<String, AtomicInteger> CACHE_MAP = new ConcurrentHashMap<>();
    private static final String KEY = "test";

    private static class TestThread implements Runnable{
        @Override
        public void run() {
            if(CACHE_MAP.get(KEY)==null){
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                CACHE_MAP.put(KEY, new AtomicInteger());
            }
            CACHE_MAP.get(KEY).incrementAndGet();
        }
    }

    public static void main(String[] args) {
        new Thread(new TestThread()).start();
        new Thread(new TestThread()).start();
        new Thread(new TestThread()).start();
        try {
            Thread.sleep(800);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("次数:"+CACHE_MAP.get(KEY).get());
    }
}
```
运行结果：

```
次数:2
```
次数并不为3 ，是因为ConcurrentHashMap的put方法跟普通的HashMap没什么区别，如果key相同，依然会覆盖。 

ConcurrentHashMap的put方法源码：

```
public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject
             (segments, (j << SSHIFT) + SBASE)) == null) 
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```
s.put(key, hash, value, ==false==) 

false 代表会覆盖。

---

### . blockqueue实现消费者和生产者通过put和take方法，介绍put和take底层实现
答：

&emsp;BlockingQueue 方法以四种形式出现，对于不能立即满足但可能在将来某一时刻可以满足的操作，这四种形式的处理方式不同：第一种是抛出一个异常，第二种是返回一个特殊值（null 或 false，具体取决于操作），第三种是在操作可以成功前，无限期地阻塞当前线程，第四种是在放弃前只在给定的最大时间限制内阻塞。下表中总结了这些方法：
![image](C:\Users\Administrator\Desktop\图片\笔记\面试题\TIM截图20180814171806.png)
- put:

将指定元素插入此队列中，将等待可用的空间.通俗点说就是>maxSize 时候，阻塞，直到能够有空间插入元素
```
public void put(E e) throws InterruptedException {  
       if (e == null) throw new NullPointerException();  
       final E[] items = this.items;  
       final ReentrantLock lock = this.lock;  
       lock.lockInterruptibly();  
       try {  
           try {  
               while (count == items.length)  
                   notFull.await();  
           } catch (InterruptedException ie) {  
               notFull.signal(); // propagate to non-interrupted thread  
               throw ie;  
           }  
           insert(e);  
       } finally {  
           lock.unlock();  
       }  
   }  
```
- take:

获取并移除此队列的头部，在元素变得可用之前一直等待 。queue的长度 == 0 的时候，一直阻塞

```
public E take() throws InterruptedException {  
    final ReentrantLock lock = this.lock;  
    lock.lockInterruptibly();  
    try {  
        try {  
            while (count == 0)  
                notEmpty.await();  
        } catch (InterruptedException ie) {  
            notEmpty.signal(); // propagate to non-interrupted thread  
            throw ie;  
        }  
        E x = extract();  
        return x;  
    } finally {  
        lock.unlock();  
    }  
} 
```
---


### . countdownlatch和CyclicBarrier底层实现原理
答：

&emsp;首先看一下jdk注释的第一句话简单阐明二者各自的意思:

```
CountDowLatch
A synchronization aid that allows one or more threads to wait until a set of operations 
being performed in other threads completes

CyclicBarrier
A synchronization aid that allows a set of threads to all wait for each other to reach 
a common barrier point
```
简单的意思就是CountDownLatch(下文用CDL)用于同步时一个或多个线程等待其他线程的某些操作完后才继续进行,而CyclicBarrier(下文用CB)用于若干线程需要阻塞在一个地方,达到指定数量后再同时进行. 

- 等待多线程完成的 CountDownLatch

CountDownLatch 允许一个或多个线程等待其他线程完成操作。

用法上CDL初始化必须带有一个count,表示需要countDown方法需要调用几次后才能接着进行. 
主要的api有: 
countDown(). 用来其他线程调用,没调用一次count–,直到为0后,CDL作用消失,线程继续进行

await().用于线程设置在哪个地方设置阻塞等待,类似对象的wait()函数.

所以基本的用法伪代码如下:

代码来自：https://blog.csdn.net/ruanjian1111ba/article/details/57150913

```
CDL latch = new CDL(2); 
thread A { 
…… 
latch.await(); 
……. 
} 
thread B { 
…… 
latch.countDown(); 
…… 
} 
thread C { 
…… 
latch.countDown(); 
…… 
} 
```
这样在thread B ,C都调用过countDown()后,A才能继续进行. 
然后,我们通过看一下源码来理解一下其中的过程.

```
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```
构造方异法里面我们看到其实是实现了内部类Sync

```
Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```
我们通过Sync的方法来看await()源码:

```
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
 }
```
这里sync调用的方法其实是父类AQS里面的方法,
```
   public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```
我们看到其实这里还是通过调用tryAcquireShared 方法实现的,而在AQS 里面tryAcquireShared() 方法还是需要子类实现的,也就是最后调用的就是Sync 里面的tryAcquireShared,我们回到Sync 源码里面。

这个Sync 在构造函数里面设置线程state 就是count , 
再看tryAcquireShared() 方法,里面的形参acquires 其实根本没用到,只是判断了state 即count 数等于0,也就是这里线程state 其实没有原来的意思了,只是用于count 计数而已. 
通过上面acquireSharedInterruptibly() 方法我们可以看出,只要 
tryAcquireShared 大于0(即count 还不是0)的话CDL的await()方法就没有阻塞,真正的阻塞实现是在doAcquireSharedInterruptibly(1)里面。

分析完await()方法后,我们来看一下countDown():
```
public void countDown() {
        sync.releaseShared(1);
    }
```
这里调用的依旧是sync父类里面的方法.在父类AQS里面的实现如下:
```
 public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
我们看到其实核心还是调用sync里面tryReleaseShared, 
回到sync里面的实现
```
 protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState(); (1)
                if (c == 0) (2)
                    return false; (3)
                int nextc = c-1; (4)
                if (compareAndSetState(c, nextc)) (5)
                    return nextc == 0; (6)
            }
        }
```
这里没有使用锁来实现,而是通过CAS(即比较交换)算法来实现,CAS的具体实现是调用更底层的UnSafe类,我们只要知道这是一种不通过锁来实现同步的方法,所以相对比较高效。

到这里为止,CDL的主要方法和原理都分析完了,我们看到其内部实现完全依靠Sync,也就是AQS,但这里Sync的实现也非常简练易懂。

- 同步屏障 CyclicBarrier

CyclicBarrier 的字面意思是可循环使用的屏障。他要做的事情是，让一组线程到达一个屏障（同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续执行。

虽然CB和CDL意思有些相似,但是实现上确不一样. 
CB同样实现了一个内部类Generation,意为一代,但是主要功能还是通过reentranLock实现的. 
先来看看CB源码里面定义的几个变量:
```
 private final ReentrantLock lock = new ReentrantLock();
    private final Condition trip = lock.newCondition();
    private final int parties;
    private final Runnable barrierCommand;
    private Generation generation = new Generation();
    private int count;
```
lock用来加锁 

trip这里英文不是旅游的意思,而是绊倒的意思,也许正好和Barrier(栅栏)相对应吧,栅栏几个栏杆挡着不给你走,来一个线程绊倒一个栏杆,倒完了路就通了。

parties 英文意党羽,代表这里需要挡住的线程数量,这个值在初始化后就不会变了。

barrierCommand:栅栏通了后就会执行这个命令。

generation意为一代,代表此刻这个栅栏 

count 线程的数量,初始化时和parties相等

再看CB的两个构造函数:
```
 public CyclicBarrier(int parties) {
        this(parties, null);
    }
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```
我们主要看第二个构造函数,初始化时parties赋值给了count,我们上面说的parties不会变的原因是程序最后修改的都是count.

再来看CB的主要几个方法: 
getPaties()获取要拦的线程的数量,返回的parties,也就是初始化的数量.

isBroken()判断栅栏是否坏了,是通过lock锁实现的,返回的是Generation类里面唯一的变量broken,同时我们看看Generation类的定义,非常简单:
```
 private static class Generation {
        boolean broken = false;
    }
```
这个内部类就持有一个bool变量,代表这个栅栏有没有坏掉(因为某些异常).

reset() 重置栅栏,意思就是一切重新开始了,之前拦截的都不算数了.这也和CB的名字Cyclic(循环)相符,也是CB的最大特点,她是可以一直重复使用的.reset方法同样用了lock锁,核心调用了两个私有方法,breakBarrier()和nextGeneration() (破坏栅栏和生成新的栅栏)
```
 public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }
```
我们来看看这俩个私有方法的实现:
```
private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
   private void nextGeneration() {
        trip.signalAll();
        count = parties;
        generation = new Generation();
    }
```
breakBarrier首先设置generation为坏掉的状态,然后重置count为初始化数量,最后通过trip这个Condition来通知所有的线程这个栅栏坏了。 

nextGeneration用在代表目前的栅栏坏掉或者目的达成了后,重新开始设置新的栅栏,先通过trip所有线程栅栏已经可以通过了,然后重置count,生成新的栅栏。

最后我们看看核心的方法也就是用来拦住线程的await方法,这个方法有俩种 ,一个带有时间限制,一个不带,但是都只是调用了私有方法dowait,所以我们只要看dowait方法就行了。
```
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken) (1)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) { (2)
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;(3)
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();(4)
                    return 0;
                } finally {
                    if (!ranAction) (5)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {(6)
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {(7)
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();(8)
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```
核心依然是用来lock锁来阻塞线程。

我们先来看看这句代码 final Generation g = generation;讲这个意思是jdk很多地方都在方法内部通过final局部变量的方式引用全局变量,这样做的好处就是一是提高效率,二是对垃圾回收也有好处(不过这里的确需要一个g的局部变量,后面会讲到),这样的方式我看到了很多次,也算一种编程技巧吧。

接下来的(1) (2)两处意思是如果栅栏坏了或者当前线程中断了都会导致栅栏失效,一切重来.(3)处的意思是每一个线程调用到await()时,count都会减少1,当减到0时(栅栏任务完成),接下来判断了是否有command命令需要执行.这些都完成后执行(4)处nextGeneration()重新生成栅栏。

(5)处代表了在command运行的时候发生了某些异常,导致没有正常生成nextGeneration,那么就破坏栅栏,重新来过。

最后我们看(6)处,这里通过一个for死循环来进行线程的阻塞判断操作,先是判断有没有设置时间限制,在线程阻塞时遇到InterruptedException(中断异常),捕获异常后里面有个if/else, 
这里分为两种情况,一种是当前线程在await的时候出现了中断,一种是别的线程出现了中断操作。

1. 当是别的线程中断时,在这个时候栅栏还没有生成新的generation且没坏,那么就需要当前线程破坏栅栏了。
2. 当是自己的中断,异常捕获后,中断标记会被清除,所以需要再次主动中断恢复中断标记,这里jdk里面还专门做了解释,并说是属于后来的执行动作,也就是说是为了通知其他线程捕获异常来破坏栅栏。

最后依然是对栅栏进行一些异常情况的判断。