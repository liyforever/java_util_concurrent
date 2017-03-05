JDK1.8 JUC源码分析(5) ReentrantReadWriteLock
=====

##功能介绍

* ReentrantReadWriteLock提供了读写锁机制，写锁使用AQA的独占模式，读锁共享模式
* 同ReetrantLock一样，ReentrantReadWriteLock也提供了公平/非公平的版本

代码分析
=====
* ReentrantReadWriteLock实现了

`java代码`

```
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}

```

* 先从构造函数看起

`java代码`
```
    /**
     * 默认使用的非公平的版本
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```

ReentrantReadWriteLock实现了内部也是基于AQA来实现的，看看公平/非公平父类的同步机制

`java代码`

```
	/**
     * Synchronization implementation for ReentrantReadWriteLock.
     * Subclassed into fair and nonfair versions.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 6317671515068378041L;

        /*
         * AQA中的状态(state)逻辑上被分为2个unsigned shorts来使用，第8位
         * 表示写锁的重入次数，高8位表示读锁的重入次数.
         */

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

分别从读锁，写锁的请求示范分析

前面说过AQA开放独占，共享模式给子类实现

* 写锁部分

这里先看写锁的请求，需要实现AQA中的tryAcquire方法

`java代码`

```
       protected final boolean tryAcquire(int acquires) {
            /*
             * 请求流程:
             * 1. 如果已有线程持有读锁，或已有线程持有写锁且持有者不是当前
	         *    线程，请求失败.
             * 2. 如果写锁重入次数上溢(超过unsignal unsigned).请求失败
             * 3. 否者,如果当前请求是一个重入请求，或队列策略允许，且CAS设置
             *    状态位成功。那么设置写锁的所有者
             */
            Thread current = Thread.currentThread();
            int c = getState();
			//获取写锁的重入次数
            int w = exclusiveCount(c);
            if (c != 0) {
                // c!=0&&w==0说明被读锁被持有，那么请求失败
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
				//运行到这里说明重入请求，判断下是否上溢，更新重入次数
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
			/*
			 * 说明当前写锁和读锁没有被持有，判断是否需要阻塞
			 * writerShouldBlock这个方法用来实现是否公平锁,
			 * 如果不需要阻塞CAS抢占写锁成功后设置写锁所有者
			 */ 
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }

```

看请求过程中使用的writerShouldBlock方法,可以看到开放给子类用以实现公平/非公平版本的锁

`java代码`

```
 		/**
         * Returns true if the current thread, when trying to acquire
         * the write lock, and otherwise eligible to do so, should block
         * because of policy for overtaking other waiting threads.
         */
        abstract boolean writerShouldBlock();	
```

继续看写锁的tryRelease,前面AQA中分析过，这个方法用于独占模式的释放，此处的语义是释放写锁

`java代码`

```
        protected final boolean tryRelease(int releases) {
			//判断当前线程是否持有写锁 
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
			//减少重入次数
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free) //重入次数为0那么，设置写锁的持有者为空
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```
* 读锁部分

这里先看读锁的请求，需要实现AQA中的tryAcquireShared方法

`java代码`

```
		protected final int tryAcquireShared(int unused) {
            /*
             * 请求流程:
             * 1. 如果写锁被持有，请求失败.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
			/**
			 * 如果写锁被持有且持有者不是当前线程，请求失败
			 * 注:这里可以看出，写锁可以被降级为读锁
			 */ 
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
			/**
			 * 如果不需要阻塞，且读锁重入次数没有上溢，且CAS设置重入次数，
			 * 成功
			 * 这里的readerShouldBlock开放给子类用以实现公平/非公平版本
			 */ 
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) { //如果读锁重入次数为0，设置当前线程为第一个获取读锁的线程
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
			//都尝试失败，继续调用fullTryAcquireShared
            return fullTryAcquireShared(current);
        }

```

继续看fullTryAcquireShared

`java代码`

```
		/**
         * Full version of acquire for reads, that handles CAS misses
         * and reentrant reads not dealt with in tryAcquireShared.
         */
        final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
					//如果写锁持有者不是当前线程请求失败
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT) 
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {//请求成功更新持有次数
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```

再来tryReleaseShared方法

`java代码`

```
        protected final boolean tryReleaseShared(int unused) {
			//更新缓存，重入次数
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
			//cas更新状态
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```

读锁部分多个方法都使用到了ThreadLocalHoldCounter，来看下HoldCounter是实现

`java代码`

```
        /**
         * A counter for per-thread read hold counts.
         * Maintained as a ThreadLocal; cached in cachedHoldCounter
         */
        static final class HoldCounter {
            int count = 0;
            // 使用id，而不是引用，避免垃圾
            final long tid = getThreadId(Thread.currentThread());
        }

        /**
         * ThreadLocal subclass. Easiest to explicitly define for sake
         * of deserialization mechanics.
         */
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
```
ThreadLocalHoldCounter缓存了当前线程的重入读锁的次数

来看支持非阻塞的tryWriteLock和tryReadLock方法:

`java代码`

```
		/**
         * Performs tryLock for write, enabling barging in both modes.
         * This is identical in effect to tryAcquire except for lack
         * of calls to writerShouldBlock.
         */
        final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                int w = exclusiveCount(c);
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }

        /**
         * Performs tryLock for read, enabling barging in both modes.
         * This is identical in effect to tryAcquireShared except for
         * lack of calls to readerShouldBlock.
         */
        final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)
                    return false;
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }
```

上面2个方法都是非公平允许抢占，可以看除了了没有调用writerShouldBlock，readShouldBlock外，tryReadLock和tryWriteLock和tryAcquireShared(tryAcquire)一样。

Sync已经实现了基础了同步机制，现在来看公平版子类的实现:

`java代码`

```
    /**
     * Fair version of Sync
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
```

可见，结合上面同步机制基类看，这里的判断读写请求是否应该阻塞的实现就是检查AQS同步等待队列中有没有比当前线程还早的线程。

非公平版的子类

`java代码`

```
	/**
     * Nonfair version of Sync
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // 写锁可以插队
        }
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
			/**
			 * 如果AQS同步等待队列头有写请求，那么当前读请求阻塞。  
			 */
            return apparentlyFirstQueuedIsExclusive();
        }
    }
```
* 有了内部的同步机制，ReentrantReadWriteLock实现了的方法只是调用内部的同步类来实现。一些检测机制如当前锁重入次数等也是调用AQA提供的方法