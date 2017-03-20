JDK1.8 JUC源码分析(4) CopyOnWriteArraySet
=====

##功能介绍

* CopyOnWriteArraySet和CopyOnWriteArrayList类似，区别在于CopyOnWriteArraySet内的元素是唯一的
* CopyOnWriteArraySet内部是基于CopyOnWriteArrayList.可以算是适配器

代码分析
=====

有了CopyOnWriteArrayList作为内部实现，CopyOnWriteArraySet实现起来就很简单

`java代码`

```
  public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
    private static final long serialVersionUID = 5457747651344034263L;

    private final CopyOnWriteArrayList<E> al;

    /**
     * Creates an empty set.
     */
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }

    public CopyOnWriteArraySet(Collection<? extends E> c) {
        if (c.getClass() == CopyOnWriteArraySet.class) {
            @SuppressWarnings("unchecked") CopyOnWriteArraySet<E> cc =
                (CopyOnWriteArraySet<E>)c;
            al = new CopyOnWriteArrayList<E>(cc.al);
        }
        else {
            al = new CopyOnWriteArrayList<E>();
            al.addAllAbsent(c);
        }
    }

    public boolean contains(Object o) {
        return al.contains(o);
    }

    public Object[] toArray() {
        return al.toArray();
    }

	/**
	 * 这里确保元素的唯一性调用了addIfAbsent方法
	 */ 
    public boolean add(E e) {
        return al.addIfAbsent(e);
    }
...
}
```

其他方法也是基于CopyOnWriteArrayList
