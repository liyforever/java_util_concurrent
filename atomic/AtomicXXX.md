JDK1.8 JUC源码分析(1) atomic-AtomicXXX
=====

##功能介绍

* 普通变量在多线程环境下不能保证现成安全

* 原子变量用CAS保证更新时的原子性，用volatile关键字保证内存可见性(volatile修饰的变量，总是从内存中读取)

代码分析
=====

* 原子变量底层的操作，总是调用sun.misc.Unsafe类，所以内部有一个Unsafe的静态引用，Unsafe类中的方法，基本都是原生方法

`java代码`

	`private static final Unsafe unsafe = Unsafe.getUnsafe();`

* AtomicInteger内部使用int来保存值

`java代码`

`private volatile int value;`
这里使用volatile来保证可见性

看AtomicInteger类CAS的实现代码

`java代码`

```
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

* 对于value的CAS操作，就是调用Unsafe的compareAndSwapInt方法,方法的实现在
  OpenJDK\openjdk\hotspot\src\share\vm\prims\unsafe.cpp

`c++代码`

		UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject      unsafe, jobject obj, jlong offset, jint e, jint x)) 
  		  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  		  oop p = JNIHandles::resolve(obj);
  		  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  		  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
		UNSAFE_END
这里调用的是Atomic的cmpxchg方法，这个方法\OpenJDK\openjdk\hotspot\src\share\vm\runtime\atomic.cpp中。

`c++代码`

```
#ifdef TARGET_OS_FAMILY_linux 
#include "os_linux.inline.hpp"
#endif
#ifdef TARGET_OS_FAMILY_solaris
#include "os_solaris.inline.hpp"
#endif
#ifdef TARGET_OS_FAMILY_windows
# include "os_windows.inline.hpp"
#endif
#ifdef TARGET_OS_FAMILY_bsd
# include "os_bsd.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_linux_x86
# include "atomic_linux_x86.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_linux_sparc
# include "atomic_linux_sparc.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_linux_zero
# include "atomic_linux_zero.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_solaris_x86
# include "atomic_solaris_x86.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_solaris_sparc
# include "atomic_solaris_sparc.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_windows_x86
# include "atomic_windows_x86.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_linux_arm
# include "atomic_linux_arm.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_linux_ppc
# include "atomic_linux_ppc.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_bsd_x86
# include "atomic_bsd_x86.inline.hpp"
#endif
#ifdef TARGET_OS_ARCH_bsd_zero
# include "atomic_bsd_zero.inline.hpp"
#endif
```

可见cmpxchg调用取决与OS，这里看OpenJDK\openjdk\hotspot\src\os_cpu\linux_x86\vm\atomic_linux_x86.inline.hpp文件。

`c++代码`

```
inline jlong    Atomic::cmpxchg    (jlong    exchange_value, volatile jlong*    dest, jlong    compare_value) {
  bool mp = os::is_MP();
  __asm__ __volatile__ (LOCK_IF_MP(%4) "cmpxchgq %1,(%3)"
                        : "=a" (exchange_value)
                        : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                        : "cc", "memory");
  return exchange_value;
}
```

从代码可以看出在内嵌汇编的cmpxchgq来实现compareAndSwapInt，这里LOCK_IF_MP在多线程情况下，决定是否锁住总线获cpu缓存.

* 再看AtomicInteger还有一个CAS操作

`java代码`

```
    public final boolean weakCompareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```
JDK中解释是略化的CAS操作，但这里看代码实现其实和compareAndSet方法是一致的，不排查后续会修改实现，最好还是根据JDK的解释来调用

* AtomicInteger中其他函数的实现，基本都是CAS，这里只看一个方法

`java代码`

```
    public final int getAndUpdate(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            prev = get();
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSet(prev, next));
        return prev;
    }
```
这些方法采用的是标准的CAS形式进行操作，失败并进行不断重试