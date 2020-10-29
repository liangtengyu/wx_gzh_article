# Java中的OutOfMemoryError的各种情况及解决和JVM内存结构

在JVM中内存一共有3种：Heap（堆内存），Non-Heap（非堆内存） 和Native（本地内存）。

堆内存是运行时分配所有类实例和数组的一块内存区域。非堆内存包含方法区和JVM内部处理或优化所需的内存，存放有类结构（如运行时常量池、字段及方法结构，以及方法和构造函数代码）。本地内存是由操作系统管理的虚拟内存。当一个应用内存不足时就会抛出java.lang.OutOfMemoryError 异常。 

<table><tbody><tr><td><strong>问题</strong></td><td><strong>表象</strong></td><td><strong>诊断工具</strong></td></tr><tr><td>内存不足</td><td><tt>OutOfMemoryError</tt></td><td><tt>Java Heap Analysis Tool(jhat)</tt>&nbsp;<sup>[4]</sup><br><tt>Eclipse Memory Analyzer(mat)</tt>&nbsp;<sup>[5]</sup></td></tr><tr><td>内存泄漏</td><td>使用内存增长，频繁GC</td><td><tt>Java Monitoring and Management Console(jconsole)</tt>&nbsp;<sup>[6]</sup><br><tt>JVM Statistical Monitoring Tool(jstat)</tt>&nbsp;<sup>[7]</sup></td></tr><tr><td>&nbsp;</td><td>一个类有大量的实例</td><td><tt>Memory Map(jmap) - "jmap -histo"</tt>&nbsp;<sup>[8]</sup></td></tr><tr><td>&nbsp;</td><td>对象被误引用</td><td><tt>jconsole</tt>&nbsp;<sup>[6]</sup>&nbsp;或&nbsp;<tt>jmap -dump + jhat</tt>&nbsp;<sup>[8][4]</sup></td></tr><tr><td><tt>Finalizers</tt></td><td>对象等待结束</td><td><tt>jconsole</tt>&nbsp;<sup>[6]</sup>&nbsp;或&nbsp;<tt>jmap -dump + jhat</tt>&nbsp;<sup>[8][4]</sup></td></tr></tbody></table>

OutOfMemoryError在开发过程中是司空见惯的，遇到这个错误，新手程序员都知道从两个方面入手来解决：一是排查程序是否有BUG导致内存泄漏；二是调整JVM启动参数增大内存。OutOfMemoryError有好几种情况，每次遇到这个错误时，观察OutOfMemoryError后面的提示信息，就可以发现不同之处，如：

java.lang.OutOfMemoryError: Java heap space  
java.lang.OutOfMemoryError: unable to create new native thread  
java.lang.OutOfMemoryError: PermGen space  
java.lang.OutOfMemoryError: Requested array size exceeds VM limit

虽然都叫OutOfMemoryError，但每种错误背后的成因是不一样的，解决方法也要视情况而定，不能一概而论。只有深入了解JVM的内存结构并仔细分析错误信息，才有可能做到对症下药，手到病除。

#### JVM规范

JVM规范对Java运行时的内存划定了几块区域 ，有：JVM栈（Java Virtual Machine Stacks）、堆（Heap）、方法区（Method Area）、常量池（Runtime Constant Pool）、本地方法栈（Native Method Stacks），但对各块区域的内存布局和地址空间却没有明确规定，而留给各JVM厂商发挥的空间。

#### HotSpot JVM

Sun自家的HotSpot JVM实现对堆内存结构有相对明确的说明。按照HotSpot JVM的实现，堆内存分为3个代：Young Generation、Old(Tenured) Generation、Permanent Generation。众所周知，GC（垃圾收集）就是发生在堆内存这三个代上面的。Young用于分配新的Java对象，其又被分为三个部分：Eden Space和两块Survivor Space(称为From和To)，Old用于存放在GC过程中从Young Gen中存活下来的对象，Permanent用于存放JVM加载的class等元数据。详情参见[HotSpot内存管理白皮书](http://www.oracle.com/technetwork/java/javase/memorymanagement-whitepaper-150215.pdf)。堆的布局图示如下：  
![](http://liuchangit.com/wp-content/uploads/2011/05/heap.png)

 ![](https://images2015.cnblogs.com/blog/285763/201510/285763-20151022173351442-1909134869.png)

根据这些信息，我们可以推导出JVM规范的内存分区和HotSpot实现中内存区域的对应关系：JVM规范的Heap对应到Young和Old Generation，方法区和常量池对应到Permanent Generation。对于Stack内存，HotSpot实现也没有详细说明，但[HotSpot白皮书](http://www.oracle.com/technetwork/java/whitepaper-135217.html)上提到，Java线程栈是用宿主操作系统的栈和线程模型来表示的，Java方法和native方法共享相同的栈。因此，可以认为在HotSpot中，JVM栈和本地方法栈是一回事。

#### 操作系统

由于一个JVM进程首先是一个操作系统进程，因此会遵循操作系统进程地址空间的规定。32位系统的地址空间为4G，即最多表示4GB的虚拟内存。在Linux系统中，高地址的1G空间（即0xC0000000~0xFFFFFFFF）被系统内核占用，低地址的3G空间（即0×00000000~0xBFFFFFFF）为用户程序所使用（显然JVM进程运行在这3G的地址空间中）。这3G的地址空间从低到高又分为多个段；**Text**段用于存放程序二进制代码；**Data**段用于存放编译时已初始化的静态变量；**BSS**段用于存放未初始化的静态变量；**Heap**即堆，用于动态内存分配的数据结构，C语言的malloc函数申请的内存即是从此处分配的，Java的new实例化的对象也是自此分配。不同于前面三个段，Heap空间是可变的，其上界由低地址向高地址增长。**内存映射区**，加载的动态链接库位于这个区中；**Stack**即栈空间，线程的执行即是占用栈内存，栈空间也是可变的，但它是通过下界从高地址向低地址移动而增长的。 图示如下：  
![](https://images2015.cnblogs.com/blog/285763/201510/285763-20151022173540052-148760824.png)

JVM本身是由native code所编写的，所以JVM进程同样具有Text/Data/BSS/Heap/MemoryMapping/Stack等内存段。而Java语言的Heap应当是建立在操作系统进程的Heap之上的，Java语言的Stack应该也是建立操作系统进程Stack之上的。 综合HotSpot的内存区域和操作系统进程的地址空间，可以大致得到下列图示：  
![](https://images2015.cnblogs.com/blog/285763/201510/285763-20151022173640864-3375078.png)

Java线程的内存是位于JVM或操作系统的栈（Stack）空间中，不同于对象——是位于堆（Heap）中。这是很多新手程序员容易误解的地方。注意，“Java线程的内存”这个用词不是指Java.lang.Thread对象的内存，**java.lang.Thread对象本身是在Heap中分配的，当调用start()方法之后，JVM会创建一个执行单元，最终会创建一个操作系统的native thread来执行，而这个执行单元或native thread是使用Stack内存空间的**。

经过上述铺垫，可以得知，JVM进程的内存大致分为Heap空间和Stack空间两部分。Heap又分为Young、Old、Permanent三个代。Stack分为Java方法栈和native方法栈（不做区分），在Stack内存区中，可以创建多个线程栈，每个线程栈占据Stack区中一小部分内存，线程栈是一个LIFO数据结构，每调用一个方法，会在栈顶创建一个Frame，方法返回时，相应的Frame会从栈顶移除（通过移动栈顶指针）。在这每一部分内存中，都有可能会出现溢出错误。回到开头的OutOfMemoryError，下面逐个说明错误原因和解决方法（每个OutOfMemoryError都有可能是程序BUG导致，因此解决方法不包括对BUG的排查）。

**OutOfMemoryError**
--------------------

**1.java.lang.OutOfMemoryError: Java heap space**  
原因：Heap内存溢出，意味着Young和Old generation的内存不够。  
解决：调整java启动参数 -Xms -Xmx 来增加Heap内存。

堆内存溢出时，首先判断当前最大内存是多少（参数：-Xmx 或 -XX:MaxHeapSize=），可以通过命令 jinfo -flag MaxHeapSize 查看运行中的JVM的配置，如果该值已经较大则应通过 mat 之类的工具查找问题，或 jmap -histo查找哪个或哪些类占用了比较多的内存。参数-verbose:gc(-XX:+PrintGC) -XX:+PrintGCDetails可以打印GC相关的一些数据。如果问题比较难排查也可以通过参数-XX:+HeapDumpOnOutOfMemoryError在OOM之前Dump内存数据再进行分析。此问题也可以通过histodiff打印多次内存histogram之前的差值，有助于查看哪些类过多被实例化，如果过多被实例化的类被定位到后可以通过btrace再跟踪。 下面代码可再现该异常： List<String> list = new ArrayList<String>(); while(true) list.add(new String("Consume more memory!"));

**2.java.lang.OutOfMemoryError: unable to create new native thread**  
原因：Stack空间不足以创建额外的线程，要么是创建的线程过多，要么是Stack空间确实小了。  
解决：由于JVM没有提供参数设置总的stack空间大小，但可以设置单个线程栈的大小；而系统的用户空间一共是3G，除了Text/Data/BSS/MemoryMapping几个段之外，Heap和Stack空间的总量有限，是此消彼长的。因此遇到这个错误，可以通过两个途径解决：1.通过-Xss启动参数减少单个线程栈大小，这样便能开更多线程（当然不能太小，太小会出现StackOverflowError）；2.通过-Xms -Xmx 两参数减少Heap大小，将内存让给Stack（前提是保证Heap空间够用）。



在JVM中每启动一个线程都会分配一块本地内存，用于存放线程的调用栈，该空间仅在线程结束时释放。当没有足够本地内存创建线程时就会出现该错误。通过以下代码可以很容易再现该问题： \[2\] while(true){ new Thread(new Runnable(){ public void run() { try {
                Thread.sleep(60\*60\*1000);
            } catch(InterruptedException e) { }        
        }    
    }).start();
}

 

**3.java.lang.OutOfMemoryError: PermGen space**  
原因：Permanent Generation空间不足，不能加载额外的类。  
解决：调整-XX:PermSize= -XX:MaxPermSize= 两个参数来增大PermGen内存。一般情况下，这两个参数不要手动设置，只要设置-Xmx足够大即可，JVM会自行选择合适的PermGen大小。



PermGen space即永久代，是非堆内存的一个区域。主要存放的数据是类结构及调用了intern()的字符串。 List<Class<?>> classes = new ArrayList<Class<?>>(); while(true){
    MyClassLoader cl \= new MyClassLoader(); try{
        classes.add(cl.loadClass("Dummy"));
    }catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
类加载的日志可以通过btrace跟踪类的加载情况： import com.sun.btrace.annotations.\*; import static com.sun.btrace.BTraceUtils.\*;

@BTrace public class ClassLoaderDefine {

    @SuppressWarnings("rawtypes")
    @OnMethod(clazz \= "+java.lang.ClassLoader", method = "defineClass", location = @Location(Kind.RETURN)) public static void onClassLoaderDefine(@Return Class cl) {
        println("=== java.lang.ClassLoader#defineClass ===");
        println(Strings.strcat("Loaded class: ", Reflective.name(cl)));
        jstack(10);
    }

}
除了btrace也可以打开日志加载的参数来查看加载了哪些类，可以把参数\-XX:+TraceClassLoading打开，或使用参数-verbose:class（-XX:+TraceClassLoading, -XX:+TraceClassUnloading），在日志输出中即可看到哪些类被加载到Java虚拟机中。该参数也可以通过jflag的命令java -jar jflagall.jar -flag +ClassVerbose动态打开-verbose:class。

下面是一个使用了

String.intern()的例子： 

List<String\> list = new ArrayList<String\>();  
int i=0;  
while(true) list.add(("Consume more memory!"+(i++)).intern());

你可以通过以下btrace脚本查找该类调用：

import com.sun.btrace.annotations.\*;  
import static com.sun.btrace.BTraceUtils.\*;

@BTrace  
public class StringInternTrace {

    @OnMethod(clazz \= "/.\*/", method \= "/.\*/", location \= @Location(value \= Kind.CALL, clazz \= "java.lang.String", method \= "intern")) public static void m(@ProbeClassName String pcm, @ProbeMethodName String probeMethod, @TargetInstance Object instance) { println(strcat(pcm, strcat("#", probeMethod))); println(strcat(">>>> ", str(instance))); }  
}



**4.java.lang.OutOfMemoryError: Requested array size exceeds VM limit**  
原因：这个错误比较少见（试着new一个长度1亿的数组看看），同样是由于Heap空间不足。如果需要new一个如此之大的数组，程序逻辑多半是不合理的。  
解决：修改程序逻辑吧。或者也可以通过-Xmx来增大堆内存。

详细信息表示应用申请的数组大小已经超过堆大小。如应用程序申请512M大小的数组，但堆大小只有256M，这里会抛出OutOfMemoryError，因为此时无法突破虚拟机限制分配新的数组。在大多少情况下是堆内存分配的过小，或是应用尝试分配一个超大的数组，如应用使用的算法计算了错误的大小。

**5.在GC花费了大量时间，却仅回收了少量内存时，也会报出OutOfMemoryError**，我只遇到过一两次。当使用-XX:+UseParallelGC或-XX:+UseConcMarkSweepGC收集器时，在上述情况下会报错，在HotSpot GC Turning文档上有说明：  
The parallel(concurrent) collector will throw an OutOfMemoryError if too much time is being spent in garbage collection: if more than 98% of the total time is spent in garbage collection and less than 2% of the heap is recovered, an OutOfMemoryError will be thrown.  
对这个问题，一是需要进行GC turning，二是需要优化程序逻辑。

**6.java.lang.StackOverflowError**  
原因：这也内存溢出错误的一种，即线程栈的溢出，要么是方法调用层次过多（比如存在无限递归调用），要么是线程栈太小。  
解决：优化程序设计，减少方法调用层次；调整-Xss参数增加线程栈大小。

#### **7.java.lang.OutOfMemoryError: request <size> bytes for <reason>. Out of swap space?**

> 本地内存分配失败。一个应用的Java Native Interface(JNI)代码、本地库及Java虚拟机都从本地堆分配内存分配空间。当从本地堆分配内存失败时抛出OutOfMemoryError异常。例如：当物理内存及交换分区都用完后，再次尝试从本地分配内存时也会抛出OufOfMemoryError异常。

#### 8. **java.lang.OutOfMemoryError: <reason> <stack trace> (Native method)**

> 如果异常的详细信息是 <reason> <stack trace> (Native method) 且一个线程堆栈被打印，同时最顶端的桢是本地方法，该异常表明本地方法遇到了一个内存分配问题。与前面一种异常相比，他们的差异是内存分配失败是JNI或本地方法发现或是Java虚拟机发现。

**9.java.lang.OutOfMemoryError: Direct buffer memory**

　　即从Direct Memory分配内存失败，Direct Buffer对象不是分配在堆上，是在Direct Memory分配，且不被GC直接管理的空间（但Direct Buffer的Java对象是归GC管理的，只要GC回收了它的Java对象，操作系统才会释放Direct Buffer所申请的空间）。通过\-XX:MaxDirectMemorySize=可以设置Direct内存的大小。

List<ByteBuffer\> list \= new ArrayList<ByteBuffer\>();  
while(true) list.add(ByteBuffer.allocateDirect(10000000));

#### 10. **java.lang.OutOfMemoryError: GC overhead limit exceeded**

JDK6新增错误类型。当GC为释放很小空间占用大量时间时抛出。一般是因为堆太小。导致异常的原因：没有足够的内存。可以通过参数\-XX:-UseGCOverheadLimit关闭这个特性。11. **java.lang.OutOfMemoryError: request <size> bytes for <reason>. Out of swap space?**

本地内存分配失败。一个应用的Java Native Interface(JNI)代码、本地库及Java虚拟机都从本地堆分配内存分配空间。当从本地堆分配内存失败时抛出OutOfMemoryError异常。例如：当物理内存及交换分区都用完后，再次尝试从本地分配内存时也会抛出OufOfMemoryError异常。

12. **java.lang.OutOfMemoryError: <reason> <stack trace> (Native method)**

如果异常的详细信息是 <reason> <stack trace> (Native method) 且一个线程堆栈被打印，同时最顶端的桢是本地方法，该异常表明本地方法遇到了一个内存分配问题。与前面一种异常相比，他们的差异是内存分配失败是JNI或本地方法发现或是Java虚拟机发现。

