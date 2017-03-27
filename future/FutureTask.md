JDK1.8 JUC源码分析(4) FutureTask
=====

##功能介绍

* FutureTask提供了异步执行计算Callbale/过程Runnable的能力，如果为未完成任务。那么所有调用get方法的线程将会被阻塞.
* FutureTask在JDK8之前都是基于AQA来实现，JDK8不再基于AQA，根据Doug Lea的说法是应该用AtomicXFieldUpdaters更新AQA内部的状态不能将效率达到极致，所以JDK8重新模拟内部色FIFO队列。


代码分析
=====

先来看看FutureTask实现了哪些接口。首先实现了RunnableFutrue接口。

`java代码`

```
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```

RunnableFutrue又扩展了Runnbale和Futrue.

`java代码`

```
public interface Future<V> {
    /**
     * 尝试取消任务的执行，如果任务已经完成或者已经被取消或者由于某种原因
     * 无法取消，方法返回false。如果任务取消成功，或者任务开始执行之前调用
     * 了取消方法，那么任务就永远不会执行了。mayInterruptIfRunning参数决定 
     * 了是否要中断执行任务的线程。
     */
    boolean cancel(boolean mayInterruptIfRunning);
    /**
     * 判断任务是否在完成之前被取消。
     */
    boolean isCancelled();
    /**
     * 判断任务是否完成。
     */
    boolean isDone();
    /**
     * 等待，直到获取任务的执行结果。如果任务还没执行完，这个方法会阻塞。
     */
    V get() throws InterruptedException, ExecutionException;
    /**
     * 等待，在给定的时间内获取任务的执行结果。
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

因为FutrueTask没有基于AQA，内部需要自己模拟FIFO队列的实现，先看链表的结构。

`java代码`

```
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```

可以看出对比AQA内部的队列，这里的使用单链表来实现,内部的数据只有一个等待线程的指针。

继续看FutrueTask内部的结构

`java代码`

```
public class FutureTask<V> implements RunnableFuture<V> {

    /**
     * 状态机改变的可能性:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    /** The underlying callable; nulled out after running */
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;

    /**
     * 返回执行结果，或抛出执行过程中吃掉的异常
     *
     * @param s completed state value
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```

来看看获取执行结果的get方法

`java代码`

```
 public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

    
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

2个版本的get方法内部都调用到了awaitDone方法.

`java代码`

```
	private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
			/**
		     * 等待期间被中断，将线程移出等待队列，重新抛出中断
		     */ 
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {//任务结束(完成/被中断/被取消)返回状态
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // 执行中，放弃CPU时间
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)//CAS入队
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
			//阻塞线程
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }

```

来看awaiteDone中调用的removeWaiter方法

`java代码`

```
 /**
  * 移除所以被取消的节点
  */ 
 private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
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

取消任务的方法

`java代码`

```
public boolean cancel(boolean mayInterruptIfRunning) {
		//这里说明任务只有在未开始的时候才允许取消
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
			//根据参数决定是否以中断的方法取消任务
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
            finishCompletion();
        }
        return true;
    }
```

在来看cancle中调用的finishCompletion方法

`java代码`

```
/**逐个唤醒阻塞在get方法上的线程*/
 private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
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

finishCompletion中留了一个done作为钩子方法，开放给子类实现在任务完成后进行一些后续工作。

`java代码`

```
    protected void done() { }
```

来看执行任务的run方法是如何包装的

`java代码·

```
public void run() {
		//同一个任务不允许执行2次
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
					//设置异常及FutureTask状态
                    setException(ex);
                }
                if (ran)//设置返回值及FutureTask状态
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

```
来看设置异常，执行结果的方法.注:这2个方法都是protected开放给子类作为钩子方法

`java代码`

```
    protected void set(V v) {
		//设置结果更新状态
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
			//唤醒阻塞在get的线程
            finishCompletion();
        }
    }

	protected void setException(Throwable t) {
		//设置结果更新状态
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
			//唤醒阻塞在get的线程
            finishCompletion();
        }
    }

  private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
		/**
		 * 如果内部状态被标记为中断中，自旋等待被中断
		 */ 
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


最后来看下FutrueTask的构造函数

`java代码`

```
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }


    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```

这里有一个将Runnable适配成Callable的方法

`java代码`

```
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }

  static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```