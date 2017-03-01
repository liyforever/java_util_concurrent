JDK1.8 JUC源码分析(4) ReentrantLock
=====

##功能介绍

* Java提供的代码层面的锁，区别与Synchronized(JVM内置)提供的锁.相对与内置锁提供了更多功能(公平，非公平版本,支持中断，支持超时

代码分析
=====

* 先从构造函数看起

`java代码`
```
	/**
     * 默认时使用的是非公平的锁
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * 选择公平或非公平类型
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

先前提到过有了AQA很容易根据对AQA内部状态赋予不同的含义，实现各种的同步工具.来看看ReentrantLock内部的同步机制

`java代码`

```
   private final Sync sync;

    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * 开放给子类，以实现基于公平，非公平的独占锁获取方式
         */
        abstract void lock();

        /**
         * 用于支持非公平版本的tryAcquire
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
			/**
			 * 这里看出非公平版本不理会AQA内部FIFO队列的排队情况
			 * 支持尝试获取锁
			 */	
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow //上溢判断
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
			//判断解锁的是否为同一个线程
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
    }
```

可见AQA内部的状态在ReentrantLock中被定义为独占锁当前重入的次数
先看非公平版本的内部同步器

`java代码`

```
	/**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```