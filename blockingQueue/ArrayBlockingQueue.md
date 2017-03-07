JDK1.8 JUC源码分析(4) ArrayBlockingQueue
=====

##功能介绍

* ArrayBlockingQueue提供了阻塞的功能,可以用于实现消费者生存者模型


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

    /** items index for next take, poll, peek or remove */
    int takeIndex;

    /** items index for next put, offer, or add */
    int putIndex;

    /** Number of elements in the queue */
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

    /**
     * Shared state for currently active iterators, or null if there
     * are known not to be any.  Allows queue operations to update
     * iterator state.
     */
    transient Itrs itrs = null;
...
}
```

可见ArrayBlockingQueue内部的容器并没有使用ArrayList，而是使用数组来实现循环队列

