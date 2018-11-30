#### ThreadLocal是一种线程封闭技术，用于隔离线程间的数据，从而避免使用同步控制。

ThreadLocal对象通常被设计为类的私有静态类型(private static)字段，用来关联线程的某种状态。

ThreadLocal主要有四个方法:
- initialValue
- get
- set
- remove(例子中未使用)

![image](https://pic4.zhimg.com/v2-79050bc93ffeec4f61dfd052428228bf_b.jpg)

##### initialValue方法
initialValue是设计给子类重写的方法，用以返回初始化的线程内部变量。在线程第一次调用get时它会被调用，但如果在调用get之前已经调用了set为线程内部变量设过值，则该方法不会被调用。所以，如果你希望手动调用set来初始化线程内部变量，则不必重写initialValue。

通常initialValue只会被调用一次，除非手动调用remove清除了内部变量，之后又调用get方法，这时initialValue会再被调用初始化一个新的内部变量返回。

##### remove方法
remove用以移除ThreadLocal对象关联的线程内部变量，某些情况需要用它来显式地移除，以防止内存泄漏。


###### ThreadLocal机制主要由Entry、ThreadLocalMap、Thread、ThreadLocal这四个类相互协作实现的。

##### Entry
~~~java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
~~~
Entry的定义很简单，它扩展自ThreadLocal类型的WeakReference类，是一个key-value对类。key是ThreadLocal对象的弱引用，value是线程的内部变量。

Entry使用弱引用作为key目的是，希望在外部不再需要访问ThreadLocal对象时可以让GC尽快地回收对象，而不必等到线程结束后。

当GC回收ThreadLocal对象后，再通过Entry.get()获取ThreadLocal对象时返回null，这使得内部能够感知什么时候不需要再持有对value的引用，从而释放Entry对象的引用，进而释放value的引用，这时如果value在外部没有任何引用的话(通常你不应该在外部持有对value的引用)，随后被GC回收。这种感知和释放的行为发生在ThreadLocal的get、set、remove操作时。

##### Thread
Thread内部持有一个ThreadLocalMap类型引用的成员变量。
~~~java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;

/*
 * InheritableThreadLocal values pertaining to this thread. This map is
 * maintained by the InheritableThreadLocal class.  
 */ 
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
~~~
threadLocals的初始值为null，它会延迟到初次访问时才实例化，即线程首个ThreadLocal对象调用get方法时才为threadLocals创建对象。


##### ThreadLocalMap
ThreadLocalMap是为ThreadLocal而设计的hash map，内部维护着一个哈希table数组，table内保存Entry的对象，通过ThreadLocal的哈希码可索引到(哈希码需转成数组下标)。

~~~java
static class ThreadLocalMap {
    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    private Entry[] table;

    /**
     * Construct a new map initially containing (firstKey, firstValue).
     * ThreadLocalMaps are constructed lazily, so we only create
     * one when we have at least one entry to put in it.
     */
    ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
}
~~~


##### ThreadLocal
ThreadLocal是整个机制的总导演，对外，它提供使用的接口；对内，它协调类之间的相互协作。

ThreadLocal内部不会持有对线程内部变量的引用，线程内部变量的引用由Entry对象持有，而Entry对象寄存在ThreadLocalMap内的table中。

每一个ThreadLocal对象对应一个唯一的哈希码(threadLocalHashCode)，通过这个哈希码可以从ThreadLocalMap中索引出对应的Entry，从而获得线程内部变量。

这里很巧妙，ThreadLocal对象与线程内部变量之间通过Entry对象间接关联，在内部只有Entry对象持有对ThreadLocal对象的弱引用，这样当外部不再使用ThreadLocal对象后，GC能够回收ThreadLocal对象，当内部探测到ThreadLocal对象被回收后就接着释放Entry对象。

---
---

1. 通常在Java的世界里，我们不需要关心对象的释放，大部分情况下GC会自动帮我们回收。

但是如果使用ThreadLocal不当，是有可能导致内存泄漏的。

ThreadLocal释放内部变量通常在以下时机:

线程结束后
显式调用remove
在调用get、set时，如果探测到ThreadLocal对象的弱引用对象get返回null顺便释放。
所以，如果线程存活的生命周期很长，特别是和进程一样长的话，就要特别注意防止ThreadLocal引入内存泄漏的风险，在不需要再使用某个线程内部变量时记得显式调用remove清理掉。

2. InheritableThreadLocal

该类扩展了 ThreadLocal，为子线程提供从父线程那里继承的值：在创建子线程时，子线程会接收所有可继承的线程局部变量的初始值，以获得父线程所具有的值。通常，子线程的值与父线程的值是一致的；但是，通过重写这个类中的 childValue 方法，子线程的值可以作为父线程值的一个任意函数。
~~~java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    protected T childValue(T parentValue) {
        return parentValue;
    }

    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }


    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
~~~
Thread的init方法中有下面代码
~~~java
if (parent.inheritableThreadLocals != null)
	this.inheritableThreadLocals =
		ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
~~~

3. 据说在JDK1.3时并不是这样设计的（待验证）

这种"以空间换时间"的方式有个显著的好处就是 变量随线程结束而销毁了

==4. ThreadLocalMap 没有next字段，所有不会存在链表。每次put之前都会检查对应的 slot 上的key是否存在，如果不存在直接插入；否则判断 key 是否相同，如果不同则放到下一个 slot 的位置， get 时也会判断如果 key 不同则去下一个位置去取==
~~~java
/**
 * Get the entry associated with key.  This method
 * itself handles only the fast path: a direct hit of existing
 * key. It otherwise relays to getEntryAfterMiss.  This is
 * designed to maximize performance for direct hits, in part
 * by making this method readily inlinable.
 *
 * @param  key the thread local object
 * @return the entry associated with key, or null if no such
 */
private Entry getEntry(ThreadLocal key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

/**
 * Version of getEntry method for use when key is not found in
 * its direct hash slot.
 *
 * @param  key the thread local object
 * @param  i the table index for key's hash code
 * @param  e the entry at table[i]
 * @return the entry associated with key, or null if no such
 */
private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
~~~
