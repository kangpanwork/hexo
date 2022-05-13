---
title: JVM
date: 2022-05-12 14:03:35
tags: Java
---

### 运行时数据区域

HotSpot 虚拟机

![61739beeb3810e7c2ae5e30bb5980584.png](https://img.gejiba.com/images/61739beeb3810e7c2ae5e30bb5980584.png)

JAVA 1.8 移除方法区，移到了本地内存的元空间。

![5249b2be8bc23b68fc0b388a0fcd1485.png](https://img.gejiba.com/images/5249b2be8bc23b68fc0b388a0fcd1485.png)



### 程序计数器

程序计数器（Program Counter Register）是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令、分支、循环、

跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。为什么需要程序计数器？程序一般都是多线程执行，JVM多线程是通过 CPU 时间片切换来实现，如果某个线程在执行的过程中挂起，当再次获取时间片时，字节码解释器需要从挂起的位置继续执行，程序计数器可以用来记录执行位置。

特点：

1. 线程隔离
2. 内存小
3. 不会造成 OOM
4. 记录程序执行字节码的地址
5. 执行 Native 本地方法，值为空。通过 JNI 调用本地 C++、C 库来实现。



### 虚拟机栈

每个方法执行都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。下图是运行时栈帧结构。

![f333c830ac58ac1de49ccda5230081d5.png](https://img.gejiba.com/images/f333c830ac58ac1de49ccda5230081d5.png)



#### 局部变量表

局部变量表用于存放方法参数和方法内部定义的局部变量，包括各种基本数据类型（boolean、byte、char、short、int、float、long、double、对象引用 reference类型）。

局部变量表 Slot 复用对垃圾收集的影响，参考代码，启动参数添加 -verbose:gc

```java
public static void main(String[] args) {
    {
        byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    // int a = 1; // 新加一个赋值操作
    System.gc();
}
```

发现并没有被回收，因为方法里面局部变量 placeholder 的 Slot 没有被其它变量复用，即使离开了方法的作用域，它仍然是一个 GC Root，取消上面注释能正常回收或者添加参数 -verbose:gc -Xcomp，在经过 JIT 编译优化后，上面代码可以正常回收，无需添加赋值操作。



#### 操作数栈

操作数栈的每一个元素可以是任意的 Java 数据类型。当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法执行的过程中，会有各种字节码指令往操作数栈中写入和提取，也就是出站和入栈。通过 javap -c 命令可以查看 class 文件的反编译指令，可分析方法如果执行字节码指令。在程序编译完成生成 class 文件之后，可以确定局部变量表的大小，以及操作数栈的大小。



#### 动态链接

​        每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接（Dynamic Linking）。Class 文件的常量池中存有大量的符号引用，字节码中的方法调用指令就以常量池中指向方法的符号引用作为参数。这些符号引用一部分会在类加载阶段或者第一次使用的时候就转化为直接引用，这种转化称为静态解析。另一部分将在每一次运行期间转化为直接引用，这部分称为动态连接。



#### 返回地址

当一个方法开始执行后，只有两种方式可以退出这个方法。第一种是执行引擎遇到任意一个方法返回的字节码指令。另一种是方法执行过程中遇到异常，并且这个异常没有在方法体内得到处理。方法退出的过程实际上就等同于把当前栈帧出栈，因此退出时可能执行的操作有：恢复上层方法的局部变量表和操作数栈，把返回值压入调用栈帧的操作数栈中，调用 PC 计数器的值指向方法调用指令后的一条指令等。



#### 虚拟机栈溢出

如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出 StackOverflowError 异常。

```java
public class JavaVMStackSOF {
    private int stackLength = 1;
    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length: " + oom.stackLength);
            throw e;
        }
    }
}
```



VM 参数分别设置  -Xms256m -Xmx256m -Xmn128m -Xss256k 和 设置 -Xms256m -Xmx256m -Xmn128m -Xss512k，查看运行结果（stack length 变大），随着线程栈的大小越大，能够支持越多的方法调用，也就是能够存储更多的栈帧，栈帧越多越耗内存。

| 参数说明 | 含义           | 说明                                                         |
| -------- | -------------- | ------------------------------------------------------------ |
| -Xms     | 初始堆大小     | 默认（MinHeapFreeRatio / -Xminf 参数可以调整）空余堆内存小于40%时，JVM 就会增大堆直到 -Xmx 的最大限制。 |
| -Xmx     | 最大堆大小     | 默认（MaxHeapFreeRatio / -Xmaxf 参数可以调整）空余堆内存大于 70% 时，JVM 会减少堆直到 -Xms的最小限制 |
| -Xmn     | 年轻代大小     | Eden + 2 survivor space                                      |
| -Xss     | 线程的堆栈大小 |                                                              |

在 stackLeak 方法增加参数，栈帧中的本地变量表就会增加，对应的栈帧内存就越大，栈的深度反而变小。



下面代码为创建线程导致内存溢出异常，

```java
public class JavaVMStackOOM {
    private static void notStop() {
        while (true) {

        }
    }
    public void stackLeakByThread() {
        while (true) {
            Thread thread = new Thread(JavaVMStackOOM::notStop);
            thread.start();
        }
    }

    public static void main(String[] args) {
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakByThread();
    }
}
```

这个异常是由于操作系统没有足够的资源来产生这个线程造成的，虚拟机提供了参数来控制 Java 堆（Xmx）和方法区（MaxPermSize）的内存最大值。程序计数器消耗内存非常小，如果不计算虚拟机本身耗费的内存，剩下的内存就由虚拟机栈和本地方法栈来瓜分，每个线程分配的栈容量越大，线程数量就越少，建立线程就很容易耗尽内存。考虑这种情况，

- 可以重新设计系统减少线程数量

- 减少最大堆和减少栈容量来换取更多的线程。

### 堆

堆是垃圾收集器管理的主要区域，也叫 GC 堆 （Garbage Collected Heap）。如果收集器采用的是分代垃圾收集算法，则堆可细分为：新生代（Eden、Survivor），老年代。在 JDK 7版本之前，还增加了永久代 （Permanent Generation），JDK 8 版本后 PermGen 已被元空间（Metaspace）取代，使用的是直接内存（不是 JVM 管控）

![e9c92acf8383df44e7beedbe095877a5.png](https://img.gejiba.com/images/e9c92acf8383df44e7beedbe095877a5.png)



#### 堆扩展

如果以下三个条件之一为 true，那么堆的活动部分将扩展为最大值。

- 垃圾收集器未释放足够的存储空间来满足分配请求。

- 可用空间小于使用  -Xminf 参数设置的最小可用空间。缺省值为 30%。

- 垃圾回收所用时间超过使用 -Xmaxt  参数设置的最大时间阈值。缺省值为 13%。

  

#### 堆压缩

如果以下所有情况都为 true，那么在收缩前将发生压缩：

- 在此垃圾回收周期中，未完成压缩。
- 在堆末端无可用块，或者堆末端可用块的大小小于所需的收缩量的 10%。
- 在上一个垃圾回收周期中，GC 未收缩或压缩。



#### 如果调整堆大小

垃圾收集器会因以下原因调整堆大小以将占用率保持在 40% 到 70% 之间：

- 堆占用率大于 70% 会导致更频繁的 GC 周期，致使性能降低。您可以通过设置 -Xminf  选项来变更此行为。
- 堆占用率小于 40% 意味着不频繁的 GC 周期。但是，这些周期会超过必需时间而导致更长的暂停时间，致使性能降低。您可以通过设置  -Xmaxf  选项来变更此行为。

在无负载但又具有压力的情况下运行应用程序时，您可以使用  -verbose:gc （垃圾收集时的信息打印 -XX:+PrintGC） 来帮助您设置初始堆大小 -Xms 和最大堆大小 -Xmx 。

#### 堆溢出

VM 参数：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError

```java
public class HeapOOM {
    static class OOMObjct {

    }

    public static void main(String[] args) {
        List<OOMObjct> list = new ArrayList<>();
        while (true) {
            list.add(new OOMObjct());
        }
    }
}
```

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid39288.hprof ...
Heap dump file created [28183701 bytes in 0.169 secs]
```

得到 Dump 文件，利用  jhat -J-Xmx1024m dump文件 命令分析，jhat 命令实际上会启动一个 JVM 来执行, 通过 -J 可以在启动 JVM 时传入一些启动参数，默认端口 7000。通过内存映像分析工具判断是内存溢出（系统没有足够的内存分配使用）还是内存泄露（程序分配的对象没有及时回收，或者永久不回收，无法释放已申请的内存空间，这些一般都是 GC 可达但是无用的对象）。内存泄露和内存溢出都会导致应用程序运行出现问题，性能下降或挂起。内存泄露是导致内存溢出的原因之一，内存泄露积累起来将导致内存溢出。内存泄露可以通过完善代码来避免，内存溢出可以通过调整配置来减少发生频率，但无法彻底避免。

如果是内存泄露，通过工具查看泄露对象到 GC Roots 的引用链，分析出泄露对象是通过怎么样的路径与 GC Roots 相关联并导致垃圾收集器无法自动回收。掌握了泄露对象的类型信息及 GC Root 引用链信息，可以定位到泄露代码的位置。

如果是内存溢出，那就说明对象可用，必须存活。应当检查 VM 的参数（-Xmx 与 -Xms），机器物理内存是否可以调大，代码检查是否存在某些对象生命周期过长，持有状态时间过长。



#### 对象在堆区的生命周期

对象优先在 Eden 区分配，当 Eden 区没有足够的空间进行分配时，VM 将发起一次 Minor GC。参考代码，设置 VM 参数  -verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8

 -XX:+PrintGCDateStamps -Xloggc:gc.log

- -XX:PrintGCTimeStamps：打印 GC 具体时间；
- -XX:PrintGCDetails ：打印出 GC 详细日志；
- -Xloggc: path：GC 日志生成路径。

```java
private static final int _1MB = 1024 * 1024;
public static void test() {
    byte[] b1, b2, b3, b4;
    b1 = new byte[2 * _1MB];
    b2 = new byte[2 * _1MB];
    b3 = new byte[2 * _1MB];
    // 出现一次 Minor GC
    b4 = new byte[4 * _1MB];
}
```

配置新生代 10M（Eden 8M，From Survivor 1M，To Survivor 1M），老年代 10M，新生代总可用空间为 9M，

Eden区 + 1个 Survivor 区的总容量。三个 2M 的对象，1个 4M 的对象，当 Eden 区占用 6M 的时候无法分配 4M的内存给 b4，触发一次 Minor GC，而三个 2M 的对象无法放入 Survivor 空间，通过分配担保机制提前移动到老年代去。GC 结束后，Eden 占 4M，Survivor 空闲，老年代占 6M。

![0bca81fb264d5dc53219946b5f1e9024.png](https://img.gejiba.com/images/0bca81fb264d5dc53219946b5f1e9024.png)



通过配置 -XX:PretenureSizeThreshold 设置大对象的阀值，例如：-XX:PretenureSizeThreshold=3145728，当对象大于 3M 的时候直接分配到老年代，减少新生代的垃圾回收。

通过配置 -XX:MaxTenuringThreshold 设置对象年龄计数器，当年龄到达设置的阀值，直接分配到老年代。（当 Eden 空间不足时，VM 触发 Minor GC，存活的对象转移到 Survivor区，对象年龄 +1，Survivor区的对象也会经历 Minor GC，对象年龄 +1）。

为了能更好的适应不同程序的内存状况，VM 不是永远地要求对象的年龄必须达到设置的年龄值才能晋升老年代，如果在 Survivor 空间中的相同年龄所有对象大小的总和大于 Survivor 空间的一半，年龄大于或者等于这些对象的对象将直接进入老年代。（动态对象年龄判断机制）

#### 内存分配调优

1. 压测工具进行压测（eg：ab 工具），分析请求接口吞吐量、并发连接数、响应时间等。

2. 分析 GC 日志，通过设置 VM配置参数 -XX:+HeapDumpOnOutOfMemoryError ，或者通过 jmap 命令获取 dump文件 。频繁的 GC 将会引起线程的上下文切换，导致系统吞吐量下降，GC 的 持续时间也会影响请求的响应时间。通过合理分配堆的新生代、老年代内存大小，减少 Minor GC 及 Full GC 频率及时长。

   说明：

   - 在设置 Eden、Survivor 区比例的时候，如果开启 AdaptiveSizePolicy，则每次 GC 后都会重新计算 Eden、From Survivor 和 To Survivor 区的大小，依据 GC 过程中统计的 GC 时间、吞吐量、内存占用量。导致SurvivorRatio 默认设置的比例会失效。在 JDK1.8 中，默认是开启 AdaptiveSizePolicy 的，通过 -XX:-UseAdaptiveSizePolicy 关闭该项配置，或显示运行 -XX:SurvivorRatio=8 将 Eden、Survivor 的比例设置为 8:2。
   - 一般先优化代码，减少大对象的数量及减少使用全局变量。

### 方法区（永久代）

用于存储已被 VM 加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。运行时常量池（Runtime Constant Pool）是方法区的一部分。Class 文件中除了类的版本、字段、方法、接口等描述信息外，还有常量池，用于存放编译期生成的各种字面量和符号引用，在类加载后存放。

#### 常量池溢出

VM 参数：-XX:Permsize=10M -XX:MaxPermSize=10M

使用 List 保持着常量池引用，避免 Full GC 回收。

```java
public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        int i = 0;
        while (true) {
            list.add(String.valueOf(i++).intern());
        }
    }
}
```

#### 方法区溢出

运行时产生大量的类去填满方法区，直到溢出。eg：利用 CGLib 直接操作字节码运行时生成大量的动态类。当前很多主流的框架，在对类进行增强时，都会使用到 CGLib 这类字节码技术，增强的类越多，需要越大的方法区空间来保证动态生成的 Class 可以加载到内存。



### 执行引擎（Execution Engine) 

主要有即时编译器（JITCompiler）和 垃圾收集（Garbage Collection）两部分组成。解释器和编译器都是将 Java 生成 .class 的字节码，解析成 cpu 所能执行的二进制指令。区别在于解释器是一行一行解释字节码指令（效率低），无需编译（编译需要很长时间），立刻执行。 jvm 启动的时候非常快，这时候用的是解释器，这样的话可以减少编译的时间，且不会出现较长的卡顿，并且随着程序运行的时间推移，即时编译时发生了作用，通过热点探测功能，将有价值的字节码编译为本地机器指令，以换取更高的程序执行效率。



垃圾回收名词解释：

- 串行回收（Serial）：单线程回收，STW 暂停所有用户线程，独占式回收。
- 并行回收（Parallel）：线程并行工作，STW，独占式回收。
- 并发回收（CMS）：GC 线程和用户线程同时进行，用户线程停顿时间很短。



#### Serial 收集器

单线程，没有线程交互的开销，消耗内存少，STW 时间长。使用 -XX:+UseSerialGC 来开启。

#### ParNew 收集器

多线程版的 Serial 收集器，新生代采用复制算法（多线程执行），老年代采用标记整理算法（单线程），在收集算法、STW、对象分配规则、回收策略等都与 serial一样。使用 -XX:UseParNewGC 来开启，使用 

-XX:ParallelGCThreads 指定线程数量，一般设置和CPU数量 一样。



#### Parallel Scavenge收集器

ParNew 和 CMS 收集器关注的是如何缩短垃圾收集时用户线程的停顿时间。Parallel Scavenge 关注的是吞吐量。
$$
吞吐量 = \frac {用户执行代码的时间}{用户执行代码的时间 + 垃圾收集的时间}
$$

#### CMS （Concurrent Mark Sweep）收集器

最短停顿时间的垃圾收集器，基于标记清除算法实现的，包括四个过程

- 初始标记：STW，速度快。
- 并发标记 ：与用户线程并发执行。
- 重新标记 ：STW，修正并发标记期间因用户继续运作而导致标记产生变动的那一部分对象的标记记录。
- 并发清除 ：与用户线程并发执行。

优点：并发收集器，STW 低停顿。
缺点：CMS 对处理器资源十分敏感，无法处理浮动垃圾，导致 Full GC 出现，内存碎片严重，会给内存分配带来巨大压力。



#### Garbage First 收集器

包括四个过程:

- 初始标记（Initial Marking）
- 并发标记（Concurrent Marking）
- 最终标记（Final Marking）
- 筛选回收（Live Data Counting and Evacuation）

G1 收集器的特定：

- 并行与并发，缩短 STW 停顿时间。
- 分代收集。
- 空间整理，标记整理算法，复制算法，不会产生内存碎片。
- 可预测的停顿。

两个收集器之间如果存在连线，就说明它们可以搭配使用，根据具体应用选择最合适的收集器。

![2f218fd75b0d81c901770faf2252bfc9.png](https://img.gejiba.com/images/2f218fd75b0d81c901770faf2252bfc9.png)



### JDK 命令行工具

- jps（JVM Process Status）: 类似 UNIX 的 ps 命令。用于查看所有 Java 进程的启动类、传入参数和 Java 虚拟机参数等信息；
- jstat（JVM Statistics Monitoring Tool）: 用于收集 HotSpot 虚拟机各方面的运行数据；
- jinfo（Configuration Info for Java） : Configuration Info for Java，显示虚拟机配置信息；
- jmap（Memory Map for Java）: 生成堆转储快照；
- jhat（JVM Heap Dump Browser） : 用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果；
- jstack（Stack Trace for Java）: 生成虚拟机当前时刻的线程快照，线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。







