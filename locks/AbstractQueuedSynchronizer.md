JDK1.8 JUC源码分析(3) AbstractQueuedSynchronizer
=====

##功能介绍

* AbstractQueuedSynchronizer(AQA)是JUC提供的基础同步类，JUC中其他同步机制(Lock、CountDownLatch)都依靠AQA进行实现

* AQA提供2中模式独占模式(用于实现Lock等),共享模式(用于实现Semaphore等),AQA内部包含一个FIFO等待队列用于管理没有获取控制权的线程

* AQA内部还提供了类似于Object类的wait，nofity，nofityAll 

* AQA内部有一个int(AbstractQueuedSynchronizer下是long)变量用于表示状态，各个同步组件可灵活运用这个变量如果Lock用这个变量表示递归锁当前的重入次数，CountDownLatch中用于表示闭锁还多少次可打开

* AQA的使用方式为作为同步组件的内部类提供同步机制

代码分析
=====

* 先来看AQA内部用的FIFO的节点结构
 
`java代码`

```
static final class Node {
        /** 标记共享模式的常量 */
        static final Node SHARED = new Node();
        /** 标记独占模式的常量 */
        static final Node EXCLUSIVE = null;

        /** 表示当前线程被取消 */
        static final int CANCELLED =  1;
        /** 表示当前节点的后续节点需要被唤醒 */
        static final int SIGNAL    = -1;
        /** 表示当前节点的线程正在等待某个条件 */
        static final int CONDITION = -2;
        /**
         * 表示当前接下来的一个acquireShared要无条件的往后续节点传递下去
         */
        static final int PROPAGATE = -3;

        /**
         * 节点状态区以下的值:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        volatile int waitStatus;

        /**
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueuing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        volatile Node prev;

        /**
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned during enqueuing, adjusted
         * when bypassing cancelled predecessors, and nulled out (for
         * sake of GC) when dequeued.  The enq operation does not
         * assign next field of a predecessor until after attachment,
         * so seeing a null next field does not necessarily mean that
         * node is at end of queue. However, if a next field appears
         * to be null, we can scan prev's from the tail to
         * double-check.  The next field of cancelled nodes is set to
         * point to the node itself instead of null, to make life
         * easier for isOnSyncQueue.
         */
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        volatile Thread thread;

        /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }


        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

    /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;

    /**
     * The synchronization state.
     */
    private volatile int state;
}

```

* 如开锁所说AQA分为独自模式和共享模式

  现在先看独占模式下的忽略终端的请求方法
    
```
	 /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  这里arg不代表任何意思,继承AQA的类可以在
     * tryAcquire中灵活运用这个值
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
 ```

   acquire首先调用tryAcquire如果返回true说明请求成功那么不阻塞当前的线程，如果返回false继续调用acquireQueued.AQA中没有实现tryAcquire方法，而是开放给子类实现，由于存在独占模式和共享模式的请求方法所以这里这个没有设计成抽象方法.

`java代码`

 ```
  /**
     * Attempts to acquire in exclusive mode. This method should query
     * if the state of the object permits it to be acquired in the
     * exclusive mode, and if so to acquire it.
     *
     * <p>This method is always invoked by the thread performing
     * acquire.  If this method reports failure, the acquire method
     * may queue the thread, if it is not already queued, until it is
     * signalled by a release from some other thread. This can be used
     * to implement method {@link Lock#tryLock()}.
     *
     * <p>The default
     * implementation throws {@link UnsupportedOperationException}.
     *
     * @param arg the acquire argument. This value is always the one
     *        passed to an acquire method, or is the value saved on entry
     *        to a condition wait.  The value is otherwise uninterpreted
     *        and can represent anything you like.
     * @return {@code true} if successful. Upon success, this object has
     *         been acquired.
     * @throws IllegalMonitorStateException if acquiring would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if exclusive mode is not supported
     */
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
 ```

请求失败的情况接下来调用acquireQueued(addWaiter(Node.EXCLUSIVE), arg)),这里先看addWaiter

`java代码`

 ```
	/**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
 ```

该方法将独占请求失败的线程添加到等待队列.这里先执行快速入队,失败或头结点还没初始化就执行一般的入队操作.

`java代码`

 ```
 	/**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
 ```

一般入队操作就是不停通过CAS操作来更新尾结点,这里只有在尾节点更新成功后才更新前驱节点,将当前线程加入到等待队列后，需要进行一些操作

`java代码`

 ```
	/**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
				/*这里只有是头结点才能再次尝试是否能获取独占模式的*/
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

 ```

这里是是要控制部分,先判断是否以独自模式请求成功，如果失败。判断当前线程是否需要堵塞.最后向上传播终端标志,继续来看shouldParkAfterFailedAcquire

`java代码`

 ```
	/**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev.
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * 这里如果当前节点的前驱节点已经正确的设置唤醒标志，
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

 ```
判断线程是否需要被阻塞,需要阻塞的线程需要将前驱节点状态位设置唤醒标志.并跳过已经被取消的节点.


* 总结非终端独占模式的请求流程
	1.  调用acquire方法
	2.  调用子类覆盖的tryAcquire尝试请求.尝试成功不阻塞当前尝试失败到步骤3
	3.  调用addWaiter将当前线程添加到内部FIFO队列
	4.  调用acquireQueued，如果当前线程的前驱节点是头结点那么允许再阻塞前再次进行独占模式下的尝试，请求成功则不阻塞当前线程，请求失败到步骤5
	5.  调用shouldParkAfterFailedAcquire当前线程是否需要被阻塞，判断是否需要阻塞依据为阻塞前需要对当前节点的前驱节点进行一些标志位的设置(对于将要阻塞的节点设置前驱节点的标志位为需要被唤醒,如果前驱节点状态为CANCELLED需要跳过这些节点，因为必然存在一个状态非CANCELLED的前驱节点)，返回true进入步骤6,返回false跳转步骤4
	6.  阻塞当前线程

这里来看释放操作的.

`java代码`

```
   /**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

这里先调用子类的tryRelease方法释放控制权,成功后，如果头结点为null说明后续没有需要唤醒的节点直接返回，如果后有节点需要调用unparkSuccessor唤醒第一个在等待的节点

`java代码`

```
   /**
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

这里尝试唤醒头结点后队列头部的节点，这个需要跳过被取消的节点.

* 总结非终端独占模式的释放
	1.  调用release方法
	2.  调用子类覆盖的tryRelease尝试请求,如果请求成功进行步骤3
	3.  如果头结点为null说明后续没有等待的节点,直接退出函数，如果非空那么调用unparkSuccessor到第4步
	4.  唤醒在FIFO第一个非取消的节点

继续看独占模式下相应中断的请求方法

`java代码`

```
 	/**
     * Acquires in exclusive interruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

代码可以看出同不响应中断的请求唯一的不同是在 不响应终端的请求在被中断后继续请求，相应中断的方法在终端后向上传播终端状态，同时调用cancelAcquire，继续看这个方法

`java代码`

```
 	/**
     * Cancels an ongoing attempt to acquire.
     *
     * @param node the node
     */
    private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;

        node.thread = null;

        // 跳过所有取消状态的前驱节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

		// predNext节点(node节点前面的第一个非取消状态节点的后继节点)是需要"断开"的节点。   
    		// 下面的CAS操作会达到"断开"效果，但(CAS操作)也可能会失败，因为可能存在其他"cancel"   
    		// 或者"singal"的竞争 .
        Node predNext = pred.next;

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        node.waitStatus = Node.CANCELLED;

        // 如果当前节点是尾节点，那么需要移除自己(将尾节点next设成null)
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
			/**
			 * 如果前驱节点不是头结点,那么需要给前驱节点设置唤醒标致(即设waitStatus域为SIGNAL
			 * 并连接当前节点的前驱节点与当前节点的后续节点
			 */ 
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
				//否则，唤醒当前节点
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```

最后带超时时间的请求

`java代码`

```
	/**
     * Acquires in exclusive timed mode.
     *
     * @param arg the acquire argument
     * @param nanosTimeout max wait time
     * @return {@code true} if acquired
     */
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
				//自旋一段时间
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

带超时的请求在失败进入阻塞之前会先自转一段时间

===

下面看共享模式的不响应中断请求

`java代码`

```
    /**
     * Acquires in shared mode, ignoring interrupts.  Implemented by
     * first invoking at least once {@link #tryAcquireShared},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquireShared} until success.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquireShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     */
    public final void acquireShared(int arg) {
		/**
		 * 这里区别于独占模式,共享模式允许多个线程请求成功(具体state含义由子类定义)
		 * 所以这里不能像独占模式一样判断布尔值.
		 */
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

   acquireShared首先调用tryAcquireShared(AQA中没有实现tryAcquire方法，而是开放给子类实现)，如果返回大于等于0说明请求成功那么不阻塞当前的线程，否则继续调用doAcquireShared.

`java代码`

```
	/**
     * Acquires in shared uninterruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
				//如果当前节点的前驱是头结点,允许再调用tryAcquireShared进行尝试
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
						//请求成功调用setHeadAndPropagate
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)//忽略中断模式不会跑到这里？
                cancelAcquire(node);
        }
    }
```

上面方法如果请求成功就调用setHeadAndPropagate,finally中的函数忽略中断的情况下不会调用,看下setHeadAndPropagate方法

`java代码`

```
 	/**
     * 将node设为FIFO的头结点，同时检测是否有处于共享模式下等待的节点，如果propagate》0或* 先前头结点是propagate，唤醒后续节点
     *
     * @param node the node
     * @param propagate the return value from a tryAcquireShared
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

接着看doReleaseShared

`java代码`

```
	/**
     * Release action for shared mode -- signals successor and ensures
     * propagation. (Note: For exclusive mode, release just amounts
     * to calling unparkSuccessor of head if it needs signal.)
     */
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
			//FIFO队列是否为空
            if (h != null && h != tail) {
                int ws = h.waitStatus;
				/**
				 * 头结点状态未SIGNAL说明需要唤醒后继节点
				 */
                if (ws == Node.SIGNAL) {
					/**
					 * 修改头节点状态，如果修改成功那么唤醒后续节点
					 */
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
				/**
				 * 如果头结点无状态,修改头结点状态为PROPAGATE
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
			// 头结点没有发生变，不需要继续检测，退出循环
            if (h == head)                   
                break;
        }
    }
```

* 总结非中断共享模式的请求过程
	1. 调用tryAcquireShared请求控制权，成功直接退出
	2. 请求失败,调用doAcquireShared,将当前线程已共享模式创建一个节点插入到FIFO队列的队尾
	3. 进入无限循环，如果当前节点的前驱节点是头结点，那么允许调用tryAcquireShared在此尝试获取控制权
	4. 如果请求成功将当前节点设为FIFO的头结点，同时检测是否需要唤醒共享模式下的后续节点.
	5. 如果第3步请求失败，那么调用shouldParkAfterFailedAcquire设置必要的状态。如果状态已经设置完成，那么阻塞当前线程。
	
* 共享模式的带中断，带超时的方法同独占模式相似都是向上传播中断异常(响应中断请求)，自旋一定时间(带超时)

来看共享模式下的释放控制权方法

`java代码`

```
    /**
     * Releases in shared mode.  Implemented by unblocking one or more
     * threads if {@link #tryReleaseShared} returns true.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryReleaseShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     * @return the value returned from {@link #tryReleaseShared}
     */
    public final boolean releaseShared(int arg) {
		/**
		 * 调用子类tryReleaseShared方法,成功后进行必要的唤醒操作
		 */
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

* AQA内部提供await和nofify实现

先来看看Conitin接口的方法定义:

`java代码`

```
public interface Condition {
    void await() throws InterruptedException;
    void awaitUninterruptibly();
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}

```

接下来看看AQA实现的类

`java代码`

```
    public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
```

可以看到内部结构非常简单,也是一个链表。这里看到对于每一个条件变量内部的队列并不是同一个.

像AQA一样，先从等待方法开始看，awati


`java代码`

```
		/**
         * Implements interruptible condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled or interrupted.
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * </ol>
         */
        public final void await() throws InterruptedException {
            if (Thread.interrupted()) //如果当前线程被中断，传播中断异常
                throw new InterruptedException();
            Node node = addConditionWaiter();//将当前线程加入到条件等待队列
			/**
             * 释放当前线程独占模式的控制权，调用await的线程必然获取到了独占模式AQA
             * 的控制权
             */ 
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
				/**
				 *
				 * 如果当前线程不再条件等待队列中，那么阻塞当前线程,其他线程调用signal(singalAll)
				 * 可能会把当前线程转移到AQA等待队列(说可能是因为signal一次只唤醒一个线程)
				 */ 
                LockSupport.park(this);
				//被唤醒后需要检查是否等待过程中被中断
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
			//重新请求AQA控制权
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
			//这里处理while中的中断
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

```

继续看把当前节点加入到条件队列的方法addConditionWaiter

`java代码`

```
 		/**
         * Adds a new waiter to wait queue.
         * @return its new wait node
         */
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // 移除被取消的节点.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null) //队列为空
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

看上面函数调用的用于断开取消节点的方法unlinkCancelledWaiters

`java代码`

```
		/**
         * Unlinks cancelled waiter nodes from condition queue.
         * Called only while holding lock. This is called when
         * cancellation occurred during condition wait, and upon
         * insertion of a new waiter when lastWaiter is seen to have
         * been cancelled. This method is needed to avoid garbage
         * retention in the absence of signals. So even though it may
         * require a full traversal, it comes into play only when
         * timeouts or cancellations occur in the absence of
         * signals. It traverses all nodes rather than stopping at a
         * particular target to unlink all pointers to garbage nodes
         * without requiring many re-traversals during cancellation
         * storms.
         */
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;//记录已经完成的节点
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
					//如果T被取消
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```

继续看await中带调用的fullyRelease方法

`java代码`

```
 	/**
     * 调用release方法传入当前的状态值，调用成功返回先前记录的状态值
     * @param node the condition node for this wait
     * @return previous sync state
     */
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

继续看await中判断是否在AQA队列中的方法isOnSyncQueue

`java代码`

```
	/**
     * Returns true if a node, always one that was initially placed on
     * a condition queue, is now waiting to reacquire on sync queue.
     * @param node the node
     * @return true if is reacquiring
     */
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
		/**
		 * 如果有后继节点说明必在AQA中，因为当前线程是获取到了AQA的独占，如果在条件队列中
		 * 必然没有后继节点
		 */
        if (node.next != null) 
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         * 反向遍历是否在AQA中
         */
        return findNodeFromTail(node);
    }

    /**
     * Returns true if node is on sync queue by searching backwards from tail.
     * Called only when needed by isOnSyncQueue.
     * @return true if present
     */
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```

继续看await中的检测while等候中是否被中断的方法checkInterruptWhileWaiting

`java代码`

```
        /** 在等待退出时重新中断(传递中断状态) */
        private static final int REINTERRUPT =  1;
        /** 在等待退出时抛出中断异常 */
        private static final int THROW_IE    = -1;

        /**
         * Checks for interrupt, returning THROW_IE if interrupted
         * before signalled, REINTERRUPT if after signalled, or
         * 0 if not interrupted.
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }

	/**
     * Transfers node, if necessary, to sync queue after a cancelled wait.
     * Returns true if thread was cancelled before being signalled.
     * 在取消等待后，将节点转移到同步队列中。如果线程在唤醒钱被 
 	 * 取消，返回true。
     * @param node the node
     * @return true if cancelled before the node was signalled
     */
    final boolean transferAfterCancelledWait(Node node) {
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            return true;
        }
        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
```

最后看下处理中断的方法

`java代码`

```
		/**
         * Throws InterruptedException, reinterrupts current thread, or
         * does nothing, depending on mode.
         */
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```

* 总结awati调用逻辑
	1. 如果当前线程有中断状态，抛出InterruptedException
	2. 将当前线程入队条件等到队列
	3. 释放AQA控制
	4. 进入条件循环，条件为判断当前线程是否在AQS同步队列中，如果不在那么阻塞当前线程；如果在AQS同步队列中，就到第7步。
    5. 当前线程被(其他线程)唤醒后，要检查等待过程中是否被中断或者取消，如果不是，继续循环，到第4步。
    6. 如果是，保存中断状态和模式，然后退出条件循环。
    7. 请求AQS控制权，然后做一些收尾工作，如果被取消，清理一下条件等待队列,然后按照中断模式处理一下中断。  

await带不响应终端，带超时的版本与前面独占模式请求累死

继续看唤醒单个线程的方法signal

`java代码`

```
        /**
         * Moves the longest-waiting thread, if one exists, from the
         * wait queue for this condition to the wait queue for the
         * owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        public final void signal() {
			//判断是否持有独占
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

       /**
         * Removes and transfers nodes until hit non-cancelled one or
         * null. Split out from signal in part to encourage compilers
         * to inline the case of no waiters.
         * @param first (non-null) the first node on condition queue
         */
        private void doSignal(Node first) {
            do {
				//唤醒第一个等待的节点
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
			//转移头结点到AQA队列
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

最后看下唤醒所有阻塞在条件队列的方法signalAll

`java代码`

```
       /**
         * Moves all threads from the wait queue for this condition to
         * the wait queue for the owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first); //此处与signal不同
        }

		/**
         * 将所有阻塞在条件变量队列的节点转移到AQA
         * @param first (non-null) the first node on condition queue
         */
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```