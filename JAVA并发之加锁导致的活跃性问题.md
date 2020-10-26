首先提及一下前置知识:

> 1.[JAVA并发之基础概念](https://mp.weixin.qq.com/s/b8aX2DAICYQ05i1DI8SAmA) 
> 2.[JAVA并发之进程VS线程](https://mp.weixin.qq.com/s/VE1FjX3Iimz21MOOQbWBCw) 
> 3.[JAVA并发之多线程引发的问题剖析及如何保证线程安全](https://mp.weixin.qq.com/s/q07rwQAGKsxeUxzw01835Q)

​		在前三章我们讨论了`多线程并发`的优点以及如何加锁来处理并发带来的`安全性问题`

​		但是加锁也为我们带来了诸多问题 如:死锁,活锁,线程饥饿等问题 这一章我我们主要处理锁带来的问题.
首先就是最出名的死锁

### 1.死锁(Deadlock)

什么是死锁

![image-20201026192610810](/Users/lty/IdeaProjects/wx_gzh_article/pic/image-20201026192610810-3715871-3715990.png)

> ​		死锁是当线程进入无限期等待状态时发生的情况，因为所请求的锁被另一个线程持有，而另一个线程又等待第一个线程持有的另一个锁 导致互相等待。总结:多个线程互相等待对方释放锁。

例如在现实中的十字路口，锁就像红路灯指示器，一旦锁坏了，就会导致交通瘫痪。
那么该如何避免这个问题呢  

#### 死锁的解决和预防

1.超时释放锁

		>​		顾名思义,这种避免死锁的方式是在尝试获取锁的时候加一个超时时间,这就意味着,如果一个线程在获取锁的门口等待太久这个线程就会放弃这次请求,退还并释放所有已经获得的锁,再在等待一段随机时间后再次尝试,这段时间其他的线程伙伴可以去尝试拿锁.

```java
public interface Lock {
	//自定义异常类
    public static class TimeOutException extends Exception{
        public TimeOutException(String message){
            super(message);
        }
    }
	//无超时锁，可以被打断
    void lock() throws  InterruptedException;
    //超时锁，可以被打断
    void lock(long molls) throws InterruptedException,TimeOutException;
    //解锁
    void unlock();
    //获取当前等待的线程
    Collection<Thread> getBlockedThread();
    //获取当前阻塞的线程数目
    int getBlockSize();
}


```

```java
public class BooleanLock  implements Lock{
    private boolean initValue;
    private Thread currenThread;
    public BooleanLock(){
        this.initValue = false;
    }
    private Collection<Thread> blockThreadCollection = new ArrayList<>();

    @Override
    public synchronized void lock() throws InterruptedException {
        while (initValue){
            blockThreadCollection.add(Thread.currentThread());
            this.wait();
        }
        //表明此时正在用，别人进来就要锁住
        this.initValue = true;
        currenThread = Thread.currentThread();
        blockThreadCollection.remove(Thread.currentThread());//从集合中删除
    }

    @Override
    public synchronized void lock(long mills) throws InterruptedException, TimeOutException {
        if (mills<=0){
            lock();
        }else {
            long hasRemain = mills;
            long endTime = System.currentTimeMillis()+mills;
            while (initValue){
                if (hasRemain<=0)
                    throw new TimeOutException("Time out");
                blockThreadCollection.add(Thread.currentThread());
                hasRemain = endTime-System.currentTimeMillis();
            }
            this.initValue = true;
            currenThread = Thread.currentThread();
        }


    }

    @Override
    public synchronized void unlock() {
        if (currenThread==Thread.currentThread()){
            this.initValue = false; //表明锁已经释放
            Optional.of(Thread.currentThread().getName()+ " release the lock monitor").ifPresent(System.out::println);
            this.notifyAll();
        }
    }

    @Override
    public Collection<Thread> getBlockedThread() {
        return Collections.unmodifiableCollection(blockThreadCollection);
    }

    @Override
    public int getBlockSize() {
        return blockThreadCollection.size();
    }
}


```

```java
public class BlockTest {
    public static void main(String[] args) throws InterruptedException {
    final BooleanLock booleanLock = new BooleanLock();
    // 使用Stream流的方式创建四个线程
        Stream.of("T1","T2","T3","T4").forEach(name->{
            new Thread(()->{
                try {
                    booleanLock.lock(10);
                    Optional.of(Thread.currentThread().getName()+" have the lock Monitor").ifPresent(System.out::println);
                    work();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (Lock.TimeOutException e) {
                    Optional.of(Thread.currentThread().getName()+" time out").ifPresent(System.out::println);
                } finally {
                    booleanLock.unlock();
                }
            },name).start();
        });
    }

  	//如果是需要一直等待就调用 lock()，如果是超时要退出来就调用超时lock(long millo)
    private static void work() throws InterruptedException{
        Optional.of(Thread.currentThread().getName()+" is 		working.....'").ifPresent(System.out::println);
        Thread.sleep(40_000);
    }
}

```

运行:

```java

T1 have the lock Monitor
T1 is working.....
T2 time out
T4 time out
T3 time out

```



2.按顺序加锁

		>按照顺序加锁是一种有效防止死锁的机制,但是这种方式,你需要先知道所有可能用到锁的位置,并对这些锁安排一个顺序

3.死锁检测

		>​		死锁检测是一个更好的死锁预防机制,主要用于超时锁和按顺序加锁不可用的场景每当一个线程获得了锁，会在线程和锁相关的数据结构中（map、graph 等等）将其记下。除此之外，每当有线程请求锁，也需要记录在这个数据结构中。当一个线程请求锁失败时，这个线程可以遍历锁的关系图看看是否有死锁发生。

如果检测出死锁，有两种处理手段：

- 释放所有锁，回退，并且等待一段随机的时间后重试。这个和简单的加锁超时类似，不一样的是只有死锁已经发生了才回退，而不会是因为加锁的请求超时了。虽然有回退和等待，但是如果有大量的线程竞争同一批锁，它们还是会重复地死锁,原因同超时类似，不能从根本上减轻竞争.
- 一个更好的方案是给这些线程设置优先级，让一个（或几个）线程回退，剩下的线程就像没发生死锁一样继续保持着它们需要的锁。如果赋予这些线程的优先级是固定不变的，同一批线程总是会拥有更高的优先级。为避免这个问题，可以在死锁发生的时候设置随机的优先级。

### 2.活锁(Livelock)

什么是活锁

![image-20201026201127219](/Users/lty/IdeaProjects/wx_gzh_article/pic/image-20201026201127219-3715883-3716018.png)

​		死锁是一直死等，活锁他不死等，它会一直执行，但是线程就是不能继续，因为它不断重试相同的操作。换句话说，就是信息处理线程并没有发生阻塞，但是永远都不会前进了,当他们为了彼此间的响应而相互礼让，使得没有一个线程能够继续前进，那么就发生了活锁

#### 避免活锁

​		解决“**活锁**”的方案很简单，谦让时，尝试等待一个随机的时间就可以了。由于等待的时间是随机的，所以同时相撞后再次相撞的概率就很低了。“等待一个随机时间”的方案虽然很简单，却非常有效，Raft 这样知名的分布式一致性算法中也用到了它。

### 3.饥饿

​		什么是饥饿

>- 高优先级线程吞噬所有的低优先级线程的 CPU 时间。
>- 线程被永久堵塞在一个等待进入同步块的状态，因为其他线程总是能在它之前持续地对该同步块进行访问。
>- 线程在等待一个本身(在其上调用 wait())也处于永久等待完成的对象，因为其他线程总是被持续地获得唤醒。

​												 ![image-20201026201620680](/Users/lty/IdeaProjects/wx_gzh_article/pic/image-20201026201620680-3715896.png)

饥饿问题最经典的例子就是哲学家问题。如图所示：有五个哲学家用餐，每个人要活得两把叉子才可以就餐。当 2、4 就餐时，1、3、5 永远无法就餐，只能看着盘中的美食饥饿的等待着。

#### 解决饥饿

Java 不可能实现 100% 的公平性，我们依然可以通过同步结构在线程间实现公平性的提高。

有三种方案：

> - 保证资源充足
> - 公平地分配资源
> - 避免持有锁的线程长时间执行

这三个方案中，方案一和方案三的适用场景比较有限，因为很多场景下，资源的稀缺性是没办法解决的，持有锁的线程执行的时间也很难缩短。倒是方案二的适用场景相对来说更多一些。
		那如何公平地分配资源呢？在并发编程里，主要是使用**公平锁**。所谓公平锁，是一种先来后到的方案，线程的等待是有顺序的，排在等待队列前面的线程会优先获得资源。

### 4.性能问题

​		并发执行一定比串行执行快吗？线程越多执行越快吗？

​		答案是：**并发不一定比串行快**。因为有创建线程和线程`上下文切换`的开销。

### 5.上下文切换

#### 什么是上下文切换？

​		当 CPU 从执行一个线程切换到执行另一个线程时，CPU 需要保存当前线程的本地数据，程序指针等状态，并加载下一个要执行的线程的本地数据，程序指针等。这个开关被称为“上下文切换”。

#### 减少上下文切换的方法

>- 无锁并发编程 - 多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些办法来避免使用锁，如将数据的 ID 按照 Hash 算法取模分段，不同的线程处理不同段的数据。
>- CAS 算法 - Java 的 Atomic 包使用 CAS 算法来更新数据，而不需要加锁。
>- 使用最少线程 - 避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这样会造成大量线程都处于等待状态。
>- 使用协程 - 在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换。

### 6.资源限制

#### 什么是资源限制

​		资源限制是指在进行并发编程时，程序的执行速度受限于计算机硬件资源或软件资源。

#### 资源限制引发的问题

​		在并发编程中，将代码执行速度加快的原则是将代码中串行执行的部分变成并发执行，但是如果将某段串行的代码并发执行，因为受限于资源，仍然在串行执行，这时候程序不仅不会加快执行，反而会更慢，因为增加了上下文切换和资源调度的时间。

#### 如何解决资源限制的问题

​		在资源限制情况下进行并发编程，根据不同的资源限制调整程序的并发度。

>- 对于硬件资源限制，可以考虑使用集群并行执行程序。
>- 对于软件资源限制，可以考虑使用资源池将资源复用。

### 总结

至本章为止,多线程并发的概念篇就结束了,实际操作篇尽情期待
持续关注公众号 `JAVA宝典`