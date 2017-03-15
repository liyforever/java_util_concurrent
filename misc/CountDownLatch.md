JDK1.8 JUC源码分析(4) CountDownLatch
=====

##功能介绍

* CountDownLatch(闭锁)提供了让1个或多个线程等待统一事件发生继续执行的功能
* 基于AQA的共享模式实现

代码分析
=====

先前提到过有了AQA很容易根据对AQA内部状态赋予不同的含义，实现各种的同步工具.来看看CountDownLatch内部的同步机制

`java代码`

```
   /**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.
     */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

		/**
		 * 闭锁没有被打开，请求总是失败
		 */ 
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

		/**
		 * CAS更新内部状态
		 */ 
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
    }
```

有了内部的同步机制实现CountDownLatch就非常简单了

`java代码`

```
	public class CountDownLatch {

    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

	/**
	 * count等于表示闭锁处于打开状态，直接返回不阻塞线程
	 */ 
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    
	/**
	 * 减少count的次数，count次数为0时唤醒所有阻塞的线程
	 */
    public void countDown() {
        sync.releaseShared(1);
    }

    public long getCount() {
        return sync.getCount();
    }
}
```