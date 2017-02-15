JDK1.8 JUC源码分析(2) atomic-AtomicXXXArray
=====

##功能介绍

* 原子类型的数组，包括AtomicIntegerArray,AtomicLongArray

代码分析
=====

* 和原子变量一样内部AtomicIntegerArray,AtomicLongArray总是调用sun.misc.Unsafe类，所以内部有一个Unsafe的静态引用，Unsafe类中的方法，基本都是原生方法

`java代码`

	`private static final Unsafe unsafe = Unsafe.getUnsafe();`

* 首先看这的静态初始方法

`java代码`

```
    private static final int base = unsafe.arrayBaseOffset(int[].class);
    private static final int shift;
    private final int[] array;

    static {
        int scale = unsafe.arrayIndexScale(int[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }
```

其中base是数组相对于数组引用第一个元素的偏移位置，如果int[] array = new int[3],其中array的地址为Ox00000000,那么第一个元素的地址为Ox0000000F,那么base值就是16。因为头部需要记录下数组大小，经历过多少次GC等一些信息.scale的值是单个数组元素的大小，这里int应该是4.通过数组引用arrary，base和scale就能唯一确定一个数组元素的内存地址，为后续一些CAS操作提供支持.

`java代码`

```
	private static long byteOffset(int i) {
        return ((long) i << shift) + base;
    }
```
这里利用位运算来计算数组偏移量，shift对于这如i = 3，那么相对与array的位置就是 3 << 2 = 3 * 4对于利用位运算是否比乘法运算？

* AtomicIntegerArray有2个构造函数

`java代码`

```
	private final int[] array;

    /**
     * Creates a new AtomicIntegerArray of the given length, with all
     * elements initially zero.
     *
     * @param length the length of the array
     */
    public AtomicIntegerArray(int length) {
        array = new int[length];
    }

    /**
     * Creates a new AtomicIntegerArray with the same length as, and
     * all elements copied from, the given array.
     * 内部数组是final保证了构造后对其他线程可见
     * @param array the array to copy elements from
     * @throws NullPointerException if array is null
     */
    public AtomicIntegerArray(int[] array) {
        // Visibility guaranteed by final field guarantees
        this.array = array.clone();
    }
```


AtomicXXXArrary的一些CAS操作，就是调用unsafe类提供的一些底层接口来实现