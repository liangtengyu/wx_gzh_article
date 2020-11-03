> [JAVA多线程并发容易引发的问题及如何保证线程安全](https://mp.weixin.qq.com/s/THRpIo9LsWyC-k6Ws36O8g)
> 之前的章节中我们介绍了在并发时,容易引发的问题及如何保证线程安全,本章节我们主讲JAVA并发中的无同步方案: `ThreadLocal`

**无同步方案**:

1.可重入代码:

> **可重入代码：**可以在代码执行的任何时刻中断它，转而去执行另外一段代码，而在控制权返回之后，原来的程序不会出现任何的错误。可重入代码有一些公共的特征，例如不依赖存储在堆上的数据和公用的系统资源、用到的状态量都由参数传入、不调用非可重入的方法等。简而言之：**如果一个方法，它的返回结果是可以预测的，只要输入了相同的数据，就能返回相同的结果，那它就满足可重入性的要求，当然也就是线程安全的。**

2.线程本地存储(ThreadLocal):

> **线程本地存储：**如果一段代码所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能保证在同一个线程中执行？如果能保证，我们就可以把共享数据的可见范围限制在同一个线程之内，这样，即是无同步也能做到避免数据争用。

 

[TOC]

### 1.ThreadLocal  介绍

> 一句话总结:
>
> **`ThreadLocal` 是一个存储在线程本地副本的工具类**,要保证线程安全，不一定非要进行同步。同步只是保证共享数据争用时的正确性，如果一个方法本来就不涉及共享数据，那么自然无须同步。既然是本地存储的，那么就只有当前线程可以访问，自然是线程安全的
>
> ![img](https://img2018.cnblogs.com/blog/1368768/201906/1368768-20190613220434628-1803630402.png)

### 2.ThreadLocal  应用

`ThreadLocal` 的常用方法：

```java
public class ThreadLocal<T> {
    public T get() {}
    public void set(T value) {}
    public void remove() {}
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {}
}
```

说明:

- `get` - 用于获取 `ThreadLocal` 在当前线程中保存的变量副本。
- `set` - 用于设置当前线程中变量的副本。
- `remove` - 用于删除当前线程中变量的副本。如果此线程局部变量随后被当前线程读取，则其值将通过调用其 `initialValue` 方法重新初始化，除非其值由中间线程中的当前线程设置。 这可能会导致当前线程中多次调用 `initialValue` 方法。
- `initialValue` - 为 ThreadLocal 设置默认的 `get` 初始值，需要重写 `initialValue` 方法 。

用法:

```java
ThreadLocal<String> threadLocal = new ThreadLocal<>();//ThreadLocal对象
threadLocal.set("java宝典");//存储内容
String str = threadLocal.get();//获取内容
```

实例:

```java
public class ThreadLocalTest {
    public static void main(String[] args) {
        ThreadLocal<Boolean> threadLocal = new ThreadLocal<>();
        threadLocal.set(false);//存数据
        printCurrentThread(threadLocal.get());
      
        new Thread(new Runnable() {
            @Override
            public void run() {
                threadLocal.set(true);//存数据
                printCurrentThread(threadLocal.get());
            }
        }, "test1").start();//线程test1
      
        new Thread(new Runnable() {
            @Override
            public void run() {
                threadLocal.set(true);//存数据
                printCurrentThread(threadLocal.get());
            }
        }, "test2").start();//线程test2
    }

    private static void printCurrentThread(boolean b) {
        System.out.println(Thread.currentThread().getName() + ":\t" + b);//打印出线程名与传的boolean值
    }
}

//result:
//main:	false
//test1:	true
//test2:	true
```

### 3.ThreadLocal  源码解析

> ThreadLocal做为数据存储类，那么关键点在于`set`与`get`方法。下面代码比较多,讲解主要在注释内.

```java
public void set(T value) {
  Thread t = Thread.currentThread();//获取当前线程
  ThreadLocalMap map = getMap(t);//获取线程的ThreadLocalMap
  if (map != null)
    map.set(this, value);//给map设置值，键为当前的ThreadLocal，值为传入的value
  else
    createMap(t, value);
}

/**
* getMap方法
*/
ThreadLocalMap getMap(Thread t) {
  return t.threadLocals;//返回传入的线程的ThreadLocalMap
}

/**
* createMap方法
*/
void createMap(Thread t, T firstValue) {
  t.threadLocals = new ThreadLocalMap(this, firstValue);//创建新的ThreadLocalMap并将值传入将线程的threadLocals设置为这个ThreadLocalMap
}

```

```java
public T get() {
  Thread t = Thread.currentThread();//获取当前线程
  ThreadLocalMap map = getMap(t);//获取当前线程的map
  if (map != null) {//如果map不为空
    ThreadLocalMap.Entry e = map.getEntry(this);//获取Entry
    if (e != null) {//Entry不为空
      @SuppressWarnings("unchecked")
      T result = (T)e.value;//获取Entry的值
      return result;//返回获取的值
    }
  }
  return setInitialValue();
}

/**
* setInitialValue方法
*/
private T setInitialValue() {
  T value = initialValue();//初始值
  Thread t = Thread.currentThread();//获取当前线程
  ThreadLocalMap map = getMap(t);//获取当前线程的map
  if (map != null)//map不为空设置默认的值，也就是null
    map.set(this, value);
  else
    createMap(t, value);//map为空新建一个map在存储默认的值
  return value;//返回默认值
}
/**
* initialValue方法
*/
protected T initialValue() {
  return null;
}

```

分析：在调用set方法时获取当前线程，通过获取当前线程的ThreadLocalMap，在map不为空的时候将值存储进去，如果map为空那么新建一个ThreadLocalMap并设置给Thread后存储传入的数据。

通过这部分源码可以看出为什么ThreadLocal只能操作自己线程里的数据，因为这里跟它线程的ThreadLocalMap有关系，再来分析ThreadLocalMap

```java
//存储数据的结构，并且是弱引用
static class Entry extends WeakReference<ThreadLocal<?>> {
  /** 与ThreadLocal关联的值 */
  Object value;

  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}

//table的初始容量，必须是2的幂
private static final int INITIAL_CAPACITY = 16;

//table用于存储数据，长度必须是2的幂
private Entry[] table;

//table中存在的数据的条目数
private int size = 0;

//阀值，用于扩容
private int threshold; // Default to 0

//阀值设置为当前传入的值的2/3倍
private void setThreshold(int len) {
  threshold = len * 2 / 3;
}

//下一个值
private static int nextIndex(int i, int len) {
  return ((i + 1 < len) ? i + 1 : 0);
}

//上一个值
private static int prevIndex(int i, int len) {
  return ((i - 1 >= 0) ? i - 1 : len - 1);
}

//构造函数
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  table = new Entry[INITIAL_CAPACITY];//初始化数组
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  table[i] = new Entry(firstKey, firstValue);//数据存储进去
  size = 1;//数组里面的数据条目数量设置为1
  setThreshold(INITIAL_CAPACITY);//这是阀值
}

```

```java
private void set(ThreadLocal<?> key, Object value) {

  // We don't use a fast path as with get() because it is at
  // least as common to use set() to create new entries as
  // it is to replace existing ones, in which case, a fast
  // path would fail more often than not.

  Entry[] tab = table;
  int len = tab.length;
  //根据threadLocalHashCode进行一个位运算（取模）得到索引i
  int i = key.threadLocalHashCode & (len-1);
 
  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();//获取当前下标Entry的值
    //如果获取的ThreadLocal相同直接替换e的值 *1*
    if (k == key) {
      e.value = value;
      return;
    }
    //如果Entry key对应的k为为null那么清空所有key为null的数据 *2*
    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }
  //如果上述都不满足，直接添加 *3*
  tab[i] = new Entry(key, value);
  int sz = ++size;
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}
//哈希值
private final int threadLocalHashCode = nextHashCode();

private static final int HASH_INCREMENT = 0x61c88647;
//返回下一个哈希值
private static int nextHashCode() {
  return nextHashCode.getAndAdd(HASH_INCREMENT);
}

```

#### 3.1解决 Hash 冲突

`ThreadLocalMap` 虽然是类似 `Map` 结构的数据结构，但它并没有实现 `Map` 接口。它不支持 `Map` 接口中的 `next` 方法，这意味着 `ThreadLocalMap` 中解决 Hash 冲突的方式并非 **拉链表** 方式。

实际上，**`ThreadLocalMap` 采用线性探测的方式来解决 Hash 冲突**。所谓线性探测，就是根据初始 key 的 hashcode 值确定元素在 table 数组中的位置，如果发现这个位置上已经被其他的 key 值占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。

```java
private Entry getEntry(ThreadLocal<?> key) {
  int i = key.threadLocalHashCode & (table.length - 1);
  Entry e = table[i];
  if (e != null && e.get() == key)
    return e;
  else
    return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
  Entry[] tab = table;
  int len = tab.length;

  while (e != null) {
    ThreadLocal<?> k = e.get();
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

```

```java
public void remove() {
  ThreadLocalMap m = getMap(Thread.currentThread());//获取当前线程的ThreadLocalMap
  if (m != null)//如果ThreadLocalMap不为空
    m.remove(this);//调用ThreadLocalMap的remove方法
}

/**
* ThreadLocalMap#remove
*/
private void remove(ThreadLocal<?> key) {
  Entry[] tab = table;
  int len = tab.length;//table的长度
  int i = key.threadLocalHashCode & (len-1);//使用哈希值取模（求余操作）
  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    if (e.get() == key) {
      e.clear();
      expungeStaleEntry(i);
      return;
    }
  }
}

/**
* ThreadLocalMap.Entry#clear
*/
public void clear() {
  this.referent = null;//值设置为null
}

/**
* ThreadLocal#expungeStaleEntry
*/
private int expungeStaleEntry(int staleSlot) {
  Entry[] tab = table;
  int len = tab.length;

  // expunge entry at staleSlot
  tab[staleSlot].value = null;
  tab[staleSlot] = null;
  size--;

  // Rehash until we encounter null
  Entry e;
  int i;
  for (i = nextIndex(staleSlot, len);
       (e = tab[i]) != null;
       i = nextIndex(i, len)) {
    ThreadLocal<?> k = e.get();
    if (k == null) {
      e.value = null;
      tab[i] = null;
      size--;
    } else {
      int h = k.threadLocalHashCode & (len - 1);
      if (h != i) {
        tab[i] = null;

        // Unlike Knuth 6.4 Algorithm R, we must scan until
        // null because multiple entries could have been stale.
        while (tab[h] != null)
          h = nextIndex(h, len);
        tab[h] = e;
      }
    }
  }
  return i;
}


```

### 4.ThreadLocal 特性

ThreadLocal和Synchronized都是为了解决多线程中相同变量的访问冲突问题，不同的点是

- Synchronized是通过线程等待，牺牲时间来解决访问冲突
- ThreadLocal是通过每个线程单独一份存储空间，牺牲空间来解决冲突，并且相比于Synchronized，ThreadLocal具有线程隔离的效果，只有在线程内才能获取到对应的值，线程外则不能访问到想要的值。

正因为ThreadLocal的线程隔离特性，使他的应用场景相对来说更为特殊一些。在android中Looper、ActivityThread以及AMS中都用到了ThreadLocal。当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用ThreadLocal。

### 5.4.ThreadLocal 内存泄露问题

`ThreadLocalMap` 的 `Entry` 继承了 `WeakReference`，所以它的 **key （`ThreadLocal` 对象）是弱引用，而 value （变量副本）是强引用**。

- 如果 `ThreadLocal` 对象没有外部强引用来引用它，那么 `ThreadLocal` 对象会在下次 GC 时被回收。
- 此时，`Entry` 中的 key 已经被回收，但是 value 由于是强引用不会被垃圾收集器回收。如果创建 `ThreadLocal` 的线程一直持续运行，那么 value 就会一直得不到回收，产生**内存泄露**。

那么如何避免内存泄漏呢？
方法就是：**使用 `ThreadLocal` 的 `set` 方法后，显示的调用 `remove` 方法** 。

```java
ThreadLocal<String> threadLocal = new ThreadLocal();
try {
    threadLocal.set("xxx");
    // ...
} finally {
    threadLocal.remove();
}
```