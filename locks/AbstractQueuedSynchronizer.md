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

