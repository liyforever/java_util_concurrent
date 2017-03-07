JDK1.8 JUC源码分析(4) ArrayBlockingQueue
=====

##功能介绍

* ArrayBlockingQueue基于数组实现有界的阻塞队列
* ArrayBlockingQueue提供了阻塞的功能,可以用于实现消费者生存者模型
* ArrayBlockingQueue支持公平/非公平策略

代码分析
=====

* 先从看内部的数据结构

`java代码`

```
public class ArrayBlockingQueue<E> extends 		AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {

    /**
     * Serialization ID. This class relies on default serialization
     * even for the items array, which is default-serialized, even if
     * it is empty. Otherwise it could not be declared final, which is
     * necessary here.
     */
    private static final long serialVersionUID = -817911632652898426L;

    /** 内部队列 */
    final Object[] items;

    /** 取元素索引 */
    int takeIndex;

    /** 存元素索引 */
    int putIndex;

    /** 元素个数 */
    int count;

    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;

        public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }

    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);

        final ReentrantLock lock = this.lock;
        lock.lock(); // Lock only for visibility, not mutual exclusion
        try {
            int i = 0;
            try {
                for (E e : c) {
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            lock.unlock();
        }
    }
...
}
```

ArrayBlockingQueue内部结构很简单，并没有使用ArrayList作为元素容器，而是使用数组来实现循环队列。加上一个递归锁和2个条件变量。公平/非公平依赖与ReentrantLock

先来看put方法

`java代码`

```
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
			/**
			  * while防止假唤醒
			  */ 
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }

   private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
		/**
         * 循环队列的操作
         */ 
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
		//唤醒阻塞在空队列的线程
        notEmpty.signal();
    }
```

`java代码`

```
	public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
			/**
			  * while防止假唤醒
			  */ 
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

	private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
		//唤醒阻塞在队列满要插入的线程
        notFull.signal();
        return x;
    }
```

其他操作类似，都是对条件变量，循环队列的操作，这里不再解析.