

ReentrantLock类的大部分逻辑，都是其均继承自AQS的内部类Sync实现的

### 啥是AQS:

Java并发编程核心在于java.concurrent.util包而juc当中的大多数同步器实现都是围绕着共同的基础行为，比如**等待队列、条件队列、独占获取、共享获取**等，而这个行为的抽象就是基于AbstractQueuedSynchronizer简称AQS 它定义了一套多线程访问共享资源的同步器框架，是一个**依赖状态(state)的同步器**。

以公平锁为例子:

```java
    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock(true);
        lock.lock(); //加锁 断点处
        try {
            Thread.sleep(5000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
          lock.unlock();
        }
    
```

#### 公平锁、非公平锁

```java
public ReentrantLock(boolean fair) { //ReetrantLock的有参构造函数
    sync = fair ? new FairSync() : new NonfairSync();
}
```



在加锁行打断点运行,  我们可以看到参数:

![image-20201105163128535](http://localhost:9998/pic/image-20201105163128535.png)

​	`记住这几个值,后面会用到.`



单步步入 F7

![image-20201105163427610](http://localhost:9998/pic/image-20201105163427610.png)

看到lock()调用了Sync实例中的lock()方法

我们可以看到Sync是在ReentrantLock类中的一个抽象内部类 继承于AbstractQueuedSynchronizer (AQS)(抽象队列同步器)
![image-20201105163720831](http://localhost:9998/pic/image-20201105163720831.png)

点击Sync类中lock抽象接口方法 我们可以发现ReetrantLock有公平和非公平锁两种方式实现方式.由于本章我们使用公平锁讲解所以我们选择公平锁的实现方式继续向下调试代码
![image-20201105165340504](http://localhost:9998/pic/image-20201105165340504.png)

```java
 /**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync { //公平锁实现方式 继承于Sync
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);//获取锁 传入参数1
        }
..
```

向下⬇️

```java
  public final void acquire(int arg) {  //acquire方法接收参数 1
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
          	//这里有两个操作 使用短路与 如果第一个操作成功第二个操作不执行
						//1.尝试以独占模式获取锁,在成功时返回 
						//2.线程进入队列排队
            selfInterrupt();
    }
```

### 方式1 首先尝试获取锁 

```java
  /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {  //参数:1
            final Thread current = Thread.currentThread();  //获取当前线程
            int c = getState();	//还记得我们第一个断点截图的state值是0
            if (c == 0) {			//是0 说明我们是第一次获取锁,如果不是0说明是重入,会进入else
                if (!hasQueuedPredecessors() &&
									//公平锁和非公平锁，主要是在方法 tryAcquire 中，是否有 !hasQueuedPredecessors() 判断。
                    compareAndSetState(0, acquires)) {//通过cas操作state更新,期望值是0,新值是1
                    setExclusiveOwnerThread(current);//设置当前拥有独占访问的线程。
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) { //如果 是重入情况 会进入这里.
										//进入前会判断是否是当前线程拿的锁
                int nextc = c + acquires; //假设重入次数为2  那么nextc = 2+1 =3
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc); //更新state值
                return true;
            }
            return false;
        }
```

先判断state是否为0，**如果为0就执行上面提到的lock方法的前半部分**，通过CAS操作将state的值从0变为1，否则判断当前线程是否为exclusiveOwnerThread，然后把state++，也就是重入锁的体现，**我们注意前半部分是通过CAS来保证同步，后半部分并没有同步的体现**，原因是：后半部分是线程重入，再次获得锁时才触发的操作，此时当前线程拥有锁，所以对ReentrantLock的属性操作是无需加锁的。**如果tryAcquire()获取失败，则要执行addWaiter()向等待队列中添加一个独占模式的节点。**

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail; // 根据初始化顺序倒序获取字段
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
//在这个判断中主要就是看当前线程是不是同步队列的首位，是：true、否：false
//这部分涉及公平锁的实现，CLH（Craig，Landin andHagersten）。三个作者的首字母组合
```

### 啥是CLH

CLH锁即Craig, Landin, and Hagersten (CLH) locks。CLH锁是一个自旋锁。能确保无饥饿性。提供先来先服务的公平性。
为什么说JUC中的实现是基于CLH的“变种”，因为原始CLH队列，一般用于实现自旋锁。而JUC中的实现，获取不到锁的线程，一般会时而阻塞，时而唤醒。

![image-20201106144400019](http://localhost:9998/pic/image-20201106144400019.png)
1.获取不到锁的线程，会进入队尾，然后自旋，直到其前驱线程释放锁,具体位置是放在tail后的null位置,并让新对象的next指向Null 
2.如果head成功拿到了锁 此时 把head中包含的线程指向为Null并将head的前任设置为head后清除现任head

### 方式2 没获取到锁,需要进入队列排队:

进入方式需要`改造项目`让多个线程进行`抢占资源`改造如下:

```java
import java.util.concurrent.locks.ReentrantLock;

class Scratch {
    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock(true);
        for (int i = 0; i < 5; i++) {
            new Thread(()->{
                lock.lock();
                try {
                    Thread.sleep(100);
                    System.out.println(Thread.currentThread().getName()+"工作结束....");
                }catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    lock.unlock();
                }
            }).start();
        }
    }
}
```

回到这里发现等待队列addWaiter方法添加了一个Null   将自己加入CLH队列的尾部 	关注公众号:[JAVA宝典]

![image-20201105171529838](http://localhost:9998/pic/image-20201105171529838.png)

发现等待队列addWaiter方法添加了一个Null  null的含义是:
![image-20201105171624412](http://localhost:9998/pic/image-20201105171624412.png)

使用null是`私有占用资源模式`,使用new Node()是 `共享模式`.(顺带一提,writelock不互斥,就是使用的共享模式)

Node还有几个等待状态:

```java
 /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        volatile int waitStatus;
```





在addWaiter(Node.EXCLUSIVE)处断点: 

```java
 /**
     * 根据给定的模式为当前节点创建一个Node
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
			//
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        //线程对应的Node
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //尾节点不为空
        if (pred != null) {
            //当前node的前驱指向尾节点
            node.prev = pred;
            //将当前node设置为新的尾节点
            //如果cas操作失败，说明线程竞争
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //lockfree的方式插入队尾
        enq(node);  //只有在 tail == null时才进入
        return node;
    }
```

先找到等待队列的tail节点pred，如果pred！=null，就把当前线程添加到pred后面进入等待队列，如果不存在tail节点执行enq()

```java
    private Node enq(final Node node) {
        //经典的lockfree算法：循环+CAS
        for (;;) {
            Node t = tail;
            //尾节点为空
            if (t == null) { // Must initialize
                //初始化头节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

这里进行了循环，**如果此时存在了tail就执行同上一步骤的添加队尾操作，如果依然不存在，就把当前线程作为head结点。**
插入节点后，调用acquireQueued()进行阻塞

```java
    /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

先获取当前节点的前一节点p，如果p是head的话就再进行一次tryAcquire(arg)操作，如果成功就返回，否则就执行**shouldParkAfterFailedAcquire、parkAndCheckInterrupt来达到阻塞效果；**

### 

### unlock

在unlock处打断点

进入了

```java
    public void unlock() {
        sync.release(1);
    }

```

![image-20201106142501425](http://localhost:9998/pic/image-20201106142501425.png)

继续跟进发现进入release方法,继续查看tryRelease(arg)方法

尝试释放锁有三种实现 我们点ReetrantLock实现方式
![image-20201106142623929](http://localhost:9998/pic/image-20201106142623929.png)

源码:
![image-20201106142942146](http://localhost:9998/pic/image-20201106142942146.png)

```java
        protected final boolean tryRelease(int releases) { //最后进入到实际解锁源码中
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread()) //判断持有线程和当前执行线程是否是同一个,否则报错
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) { //判断释放掉releases参数后是否是0 如果 为0 设置当前排他锁的占有线程为Null
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);//更新state为0
            return free;//返回解锁是否成功
        }
```

