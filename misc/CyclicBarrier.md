JDK1.8 JUC源码分析(4) CyclicBarrier
=====

##功能介绍

* CyclicBarrier(栅栏)和CountDownLatch很相似，区别在于CountDownLatch只能使用一次，而CyclicBarrier可以多次使用。
* CyclicBarrier允许在某个现成无法完成任务的时候打破。
* CyclicBarrier当所有线程都达到的时候允许执行某个动作(Runnable)

代码分析
=====

与CountDownLatch不同，CyclicBarrier不是基于AQA来实现，而是基于条件变量来实现先来看看内部的同步机制

`java代码`

```
   public class CyclicBarrier {
    /**
     * Each use of the barrier is represented as a generation instance.
     * The generation changes whenever the barrier is tripped, or
     * is reset. There can be many generations associated with threads
     * using the barrier - due to the non-deterministic way the lock
     * may be allocated to waiting threads - but only one of these
     * can be active at a time (the one to which {@code count} applies)
     * and all the rest are either broken or tripped.
     * There need not be an active generation if there has been a break
     * but no subsequent reset.
     */
    private static class Generation {
        boolean broken = false;
    }

    /** 保护栅栏的锁 */
    private final ReentrantLock lock = new ReentrantLock();
    /** 等待栅栏打开的条件变量 */
    private final Condition trip = lock.newCondition();
    /** 由于栅栏可以复用，用于记录栅栏的数量，复位时可用 */
    private final int parties;
    /* 栅栏被打破后执行的动作 */
    private final Runnable barrierCommand;
    /** The current generation */
    private Generation generation = new Generation();

    /**
     * 记录当前通过栅栏线程的数量，当栅栏被打破/重置时，被重新设置为parties
     */
    private int count;
  }
```

接下来看达到栅栏时等候其他线程的方法

`java代码`

```
	public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
```

可见所有的等待方法都调用了dowait，dowait蕴含了所有情况。

`java代码`

```
	/**
     * Main barrier code, covering the various policies.
     */
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;
			
			/**
			 * 如果栅栏被打破(等待栅栏的线程被中断\栅栏被重置)抛出
			 * BrokenBarrierException异常
			 */ 
            if (g.broken)
                throw new BrokenBarrierException();
			
			/**
			 * 线程被中断，打破栅栏.抛出中断异常 
			 */ 
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
			/**
			 * 所有线程都通过栅栏
			 */ 
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
					//可见执行命令的是最后一个达到栅栏的线程
                    if (command != null) 
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
					/**
					 * 执行到这里，说明执行command的时候出现了异常，
					 * 需要打破栅栏
					 */ 
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)//无限等待
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
					/**
					 * 等待期间栅栏没有被重置，且没有被打破
					 * 打破栅栏抛出中断异常
					 */ 
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();
				
				/**
				 * 说明等待过程栅栏被重置过，不继续等待
				 */ 
                if (g != generation)
                    return index;
				
				/**等待超时抛出超时异常**/
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