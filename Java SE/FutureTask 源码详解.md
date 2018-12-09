
```
 private volatile int state;
    private static final int NEW          = 0;  //未启动
    private static final int COMPLETING   = 1;  //已完成
    private static final int NORMAL       = 2;  //正常
    private static final int EXCEPTIONAL  = 3;  //异常  
    private static final int CANCELLED    = 4;  //被取消
    private static final int INTERRUPTING = 5;  //被打断
    private static final int INTERRUPTED  = 6;  //打断

    /** The underlying callable; nulled out after running */
    private Callable<V> callable;
    /** 返回的结果（正常结果或者异常） */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** 当前执行的线程，CAS 执行 */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;
```

初始化一个callable

```
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```
# 执行过程

run 执行

```
public void run() {
        // 已启动或CAS 置换失败，不执行run 主体
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //设置错误的结果
                    setException(ex);
                }
                if (ran)
                    //设置正常的结果
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            //如果是被打断，让线程让步
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
    
//设置正常返回
protected void set(V v) {
        // CAS 置换状态为完成
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            //状态为正常
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }

//设置异常返回
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            //状态为异常
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
    
//被打断的线程让步
private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt

        // assert state == INTERRUPTED;

        // We want to clear any interrupt we may have received from
        // cancel(true).  However, it is permissible to use interrupts
        // as an independent mechanism for a task to communicate with
        // its caller, and there is no way to clear only the
        // cancellation interrupt.
        //
        // Thread.interrupted();
    }
```
runAndReset 执行但不设置结果
```
protected boolean runAndReset() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }
```

finishCompletion结束完成，循环唤醒被阻塞的线程

```
//结束完成
//循环唤醒被阻塞的线程
private void finishCompletion() {
        // assert state > COMPLETING;
        // 死循环保证所有的阻塞线程被唤醒
        for (WaitNode q; (q = waiters) != null;) {
            //CAS 保证线程安全
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                //循环唤醒线程
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```
# 获取
带时间参数和不带时间参数的get

```
public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }
```
awaitDone 根据callable 的执行状态进行不同的处理

```
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            // 如果当前线程被打断了，清理一遍等待队列
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            //如果已经执行完返回对应状态码
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            //如果刚刚完成，让步
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            //第一个阻塞，构造等待节点
            else if (q == null)
                q = new WaitNode();
            //第一个阻塞，把waiters 设置为q，CAS 保证第一个
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            //超时返回
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            //还没启动或正在执行 加入等待队列（上面那个waiters 相当于等待头）
            else
                LockSupport.park(this);
        }
    }
```
removeWaiter 如果当前线程被打断了，清理一遍等待队列

```
private void removeWaiter(WaitNode node) {
        if (node != null) {
            //把当前线程设为null
            node.thread = null;
            retry:  //重复操作
            // 死循环
            for (;;) {          // restart on removeWaiter race
                //三节点判断线程为null 的节点， prv->q->s
                //如果为q 为null ， prv->s
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            //跳到上面retry: 重新执行
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }
```
# 返回结果

```
 private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```

# 取消

```
public boolean cancel(boolean mayInterruptIfRunning) {
        //除了当前未启动并且置换状态成功
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            //强行打断
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            //结束完成
            finishCompletion();
        }
        return true;
    }
```
