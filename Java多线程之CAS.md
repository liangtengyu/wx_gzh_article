### CAS  (Compare and Swap)

CAS字面意思为比较并交换.CAS 有 3 个操作数，分别是：内存值 M，期望值 E，更新值 U。当且仅当内存值 M 和期望值 E 相等时，将内存值 M 修改为 U，否则什么都不做。

### 1.CAS的应用场景

**CAS 只适用于线程冲突较少的情况**。

CAS 的典型应用场景是：

- 原子类
- 自旋锁

#### 1.1 原子类

> 原子类是 CAS 在 Java 中最典型的应用。

我们先来看一个常见的代码片段。

```
if(a==b) {
    a++;
}
```

如果 `a++` 执行前， a 的值被修改了怎么办？还能得到预期值吗？出现该问题的原因是在并发环境下，以上代码片段不是原子操作，随时可能被其他线程所篡改。

解决这种问题的最经典方式是应用原子类的 `incrementAndGet` 方法。

```java
public class AtomicIntegerDemo {

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        final AtomicInteger count = new AtomicInteger(0);
        for (int i = 0; i < 10; i++) {
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    count.incrementAndGet();
                }
            });
        }

        executorService.shutdown();
        executorService.awaitTermination(3, TimeUnit.SECONDS);
        System.out.println("Final Count is : " + count.get());
    }

}
```

J.U.C 包中提供了 `AtomicBoolean`、`AtomicInteger`、`AtomicLong` 分别针对 `Boolean`、`Integer`、`Long` 执行原子操作，操作和上面的示例大体相似，不做赘述。

#### 1.2 自旋锁

利用原子类（本质上是 CAS），可以实现自旋锁。

所谓自旋锁，是指线程反复检查锁变量是否可用，直到成功为止。由于线程在这一过程中保持执行，因此是一种忙等待。一旦获取了自旋锁，线程会一直保持该锁，直至显式释放自旋锁。

示例：非线程安全示例

```java
public class AtomicReferenceDemo {

    private static int ticket = 10;

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        for (int i = 0; i < 5; i++) {
            executorService.execute(new MyThread());
        }
        executorService.shutdown();
    }

    static class MyThread implements Runnable {

        @Override
        public void run() {
            while (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + " 卖出了第 " + ticket + " 张票");
                ticket--;
            }
        }

    }

}
```

输出结果：

```
pool-1-thread-2 卖出了第 10 张票
pool-1-thread-1 卖出了第 10 张票
pool-1-thread-3 卖出了第 10 张票
pool-1-thread-1 卖出了第 8 张票
pool-1-thread-2 卖出了第 9 张票
pool-1-thread-1 卖出了第 6 张票
pool-1-thread-3 卖出了第 7 张票
pool-1-thread-1 卖出了第 4 张票
pool-1-thread-2 卖出了第 5 张票
pool-1-thread-1 卖出了第 2 张票
pool-1-thread-3 卖出了第 3 张票
pool-1-thread-2 卖出了第 1 张票
```

很明显，出现了重复售票的情况。

【示例】使用自旋锁来保证线程安全

可以通过自旋锁这种非阻塞同步来保证线程安全，下面使用 `AtomicReference` 来实现一个自旋锁。

```java
public class AtomicReferenceDemo2 {

    private static int ticket = 10;

    public static void main(String[] args) {
        threadSafeDemo();
    }

    private static void threadSafeDemo() {
        SpinLock lock = new SpinLock();
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        for (int i = 0; i < 5; i++) {
            executorService.execute(new MyThread(lock));
        }
        executorService.shutdown();
    }

    static class SpinLock {

        private AtomicReference<Thread> atomicReference = new AtomicReference<>();

        public void lock() {
            Thread current = Thread.currentThread();
            while (!atomicReference.compareAndSet(null, current)) {}
        }

        public void unlock() {
            Thread current = Thread.currentThread();
            atomicReference.compareAndSet(current, null);
        }

    }

    static class MyThread implements Runnable {

        private SpinLock lock;

        public MyThread(SpinLock lock) {
            this.lock = lock;
        }

        @Override
        public void run() {
            while (ticket > 0) {
                lock.lock();
                if (ticket > 0) {
                    System.out.println(Thread.currentThread().getName() + " 卖出了第 " + ticket + " 张票");
                    ticket--;
                }
                lock.unlock();
            }
        }

    }

}
```

输出结果：

```
pool-1-thread-2 卖出了第 10 张票
pool-1-thread-1 卖出了第 9 张票
pool-1-thread-3 卖出了第 8 张票
pool-1-thread-2 卖出了第 7 张票
pool-1-thread-3 卖出了第 6 张票
pool-1-thread-1 卖出了第 5 张票
pool-1-thread-2 卖出了第 4 张票
pool-1-thread-1 卖出了第 3 张票
pool-1-thread-3 卖出了第 2 张票
pool-1-thread-1 卖出了第 1 张票
```

### 2.CAS 的原理

Java 主要利用 `Unsafe` 这个类提供的 CAS 操作。`Unsafe` 的 CAS 依赖的是 JVM 针对不同的操作系统实现的硬件指令 **`Atomic::cmpxchg`**。`Atomic::cmpxchg` 的实现使用了汇编的 CAS 操作，并使用 CPU 提供的 `lock` 信号保证其原子性。

### 3.CAS 带来的问题

一般情况下，CAS 比锁性能更高。因为 CAS 是一种非阻塞算法，所以其避免了线程阻塞和唤醒的等待时间。

但是，事物总会有利有弊，CAS 也存在三大问题：

- ABA 问题
- `循环时间长开销大`
- `只能保证一个共享变量的原子性`

如何解决这三个问题:

#### 3.1  ABA 问题

**如果一个变量初次读取的时候是 A 值，它的值被改成了 B，后来又被改回为 A，那 CAS 操作就会误认为它从来没有被改变过**。

J.U.C 包提供了一个带有标记的**原子引用类  如:`AtomicStampedReference` 来解决这个问题**，它可以通过控制变量值的版本来保证 CAS 的正确性。大部分情况下 ABA 问题不会影响程序并发的正确性，如果需要解决 ABA 问题，改用**传统的互斥同步可能会比原子类更高效**。
解决方案：增加标志位，例如：AtomicMarkableReference、AtomicStampedReference

#### 3.2  循环时间长开销大

**自旋 CAS （不断尝试，直到成功为止）如果长时间不成功，会给 CPU 带来非常大的执行开销**。

如果 JVM 能支持处理器提供的 `pause` 指令那么效率会有一定的提升，`pause` 指令有两个作用：

- 它可以延迟流水线执行指令（de-pipeline）,使 CPU 不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。
- 它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起 CPU 流水线被清空（CPU pipeline flush），从而提高 CPU 的执行效率。

解决方案：因为是while循环，消耗必然大。设置尝试次数上限

#### 3.3只能保证一个共享变量的原子性

当对一个共享变量执行操作时，我们可以使用循环 CAS 的方式来保证原子操作，但是对多个共享变量操作时，循环 CAS 就无法保证操作的原子性，这个时候就可以用锁。

或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量 `i ＝ 2, j = a`，合并一下 `ij=2a`，然后用 CAS 来操作 `ij`。从 Java 1.5 开始 JDK 提供了 `AtomicReference` 类来保证引用对象之间的原子性
解决方案：用AtomicReference把多个变量封装成一个对象来进行CAS操作.

