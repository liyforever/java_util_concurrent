JDK1.8 JUC源码分析(4) DelayQueue
=====

##功能介绍

* DelayQueue队列只允许放入到期才能取出的对象
* 所有加入DelayQueue的元素需要实现Delayed接口

代码分析
=====

* 先来看看Delayed接口，扩展了Comparable接口，可以获取元素还有多久到期

`java代码`

```
public interface Delayed extends Comparable<Delayed> {

    /**
     * Returns the remaining delay associated with this object, in the
     * given time unit.
     *
     * @param unit the time unit
     * @return the remaining delay; zero or negative values indicate
     * that the delay has already elapsed
     */
    long getDelay(TimeUnit unit);
}
```

* 内部的数据结构

`java代码`

```
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {

    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();

    /**
     * 指定等待到期元素的线程
     * 领导者追随者，尽量减少不必要的等待时间.
     * 1.所有的线程都在睡眠
     * 2.只有leader等待元素到期
     * 3.元素到期后leader取走元素，并随机唤醒一个等待的线程接替自己成功leader
     */
    private Thread leader = null;

    /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
     */
    private final Condition available = lock.newCondition();

    /**
     * Creates a new {@code DelayQueue} that is initially empty.
     */
    public DelayQueue() {}

    /**
     * Creates a {@code DelayQueue} initially containing the elements of the
     * given collection of {@link Delayed} instances.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public DelayQueue(Collection<? extends E> c) {
        this.addAll(c);
    }
...
}
```

DelayQueue内部使用有限队列作为底层容器，因为是无界队列，所以没有等待队列有空位的条件变量。并使用leader来优化等待，一次只唤醒一个线程。

先来看put方法

`java代码`

```
    **
     * Inserts the specified element into this delay queue.
     *
     * @param e the element to add
     * @return {@code true}
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
			/**
			 * 当前offer的对象，最先达到到期实现。唤醒一个线程
			 * 作为leader
			 */ 
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    /**
     * Inserts the specified element into this delay queue. As the queue is
     * unbounded this method will never block.
     *
     * @param e the element to add
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) {
        offer(e);
    }
```

继续看poll方法

`java代码`

```
	   /**
     * Retrieves and removes the head of this queue, or returns {@code null}
     * if this queue has no elements with an expired delay.
     *
     * @return the head of this queue, or {@code null} if this
     *         queue has no elements with an expired delay
     */
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            E first = q.peek();
			/**
			 * 队列为空，或没有对象到期,返回null
			 */ 
            if (first == null || first.getDelay(NANOSECONDS) > 0)
                return null;
            else
                return q.poll();
        } finally {
            lock.unlock();
        }
    }
```

最后来看下take方法

`java代码`

```
 	/**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element with an expired delay is available on this queue.
     *
     * @return the head of this queue
     * @throws InterruptedException {@inheritDoc}
     */
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)//队列为空，等待
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)//对象到期
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
							/**
							 * 等待delay时间，刚好头节点对象到期
							 */ 
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally { //如果当前没有leader,重新唤醒选择一个线程作为新的leader
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```

其他操作类似，都是对条件变量，循环队列的操作，这里不再解析.