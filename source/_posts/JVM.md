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



#### 程序计数器

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

![9f5720e95820992dffece227c332c6a1.png](https://img.gejiba.com/images/9f5720e95820992dffece227c332c6a1.png)



![b28ba1388a7bd111e0c4f74296bb0d57.png](https://img.gejiba.com/images/b28ba1388a7bd111e0c4f74296bb0d57.png)

1. 局部变量表用于存放方法参数和方法内部定义的局部变量，包括各种基本数据类型（boolean、byte、char、short、int、float、long、double、对象引用 reference类型）。

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

   

2. 操作数栈的每一个元素可以是任意的 Java 数据类型。当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法执行的过程中，会有各种字节码指令往操作数栈中写入和提取，也就是出站和入栈。通过 javap -c 命令可以查看 class 文件的反编译指令，可分析方法如果执行字节码指令。在程序编译完成生成 class 文件之后，可以确定局部变量表的大小，以及操作数栈的大小。

   

| 指令码 | 反编译指令 | 说明                                               |
| ------ | ---------- | -------------------------------------------------- |
| 0x94   | lcmp       | 比较栈顶 long 类型数据的值大小，并将结果压入栈顶。 |

方法异常或者 return 会导致栈帧弹出。



3.动态链接

4.返回地址

5.虚拟机栈溢出，如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出 StackOverflowError 异常。

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
| -Xmx     | 最大堆大小     | 默认（MaxHeapFreeRatio 参数可以调整）空余堆内存大于 70% 时，JVM 会减少堆直到 -Xms的最小限制 |
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

1. 可以重新设计系统减少线程数量
2. 减少最大堆和减少栈容量来换取更多的线程。

