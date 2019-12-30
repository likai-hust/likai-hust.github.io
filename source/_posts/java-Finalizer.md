---
title: 【翻译】通过调试来理解Finalizer
date: 2019-12-30 13:58:22
tags: [Java,Finalizer]
categories: 
- Java
- 语言基础
---

本文介绍的是Java里一个内建的概念，Finalizer。你可能对它对数家珍，但也可能从未听闻过，这得看你有没有花时间完整地看过一遍java.lang.Object类了。在java.lang.Object里面就有一个finalize()的方法。这个方法的实现是空的，不过一旦实现了这个方法，就会触发JVM的内部行为，威力和危险并存。
<!--more-->
如果JVM发现某个类实现了finalize()方法的话，那么见证奇迹的时刻到了。我们先来创建一个实现了这个非凡的finalize()方法的类，然后看下这种情况下JVM的处理会有什么不同。我们先从一个简单的示例程序开始：
```java
import java.util.concurrent.atomic.AtomicInteger;

class Finalizable {
      static AtomicInteger aliveCount = new AtomicInteger(0);

      Finalizable() {
            aliveCount.incrementAndGet();
     }

     @Override
     protected void finalize() throws Throwable {
                  Finalizable.aliveCount.decrementAndGet();
     }

      public static void main(String args[]) {
            for (int i = 0;; i++) {
                  Finalizable f = new Finalizable();
                  if ((i % 100_000) == 0) {
                        System.out.format("After creating %d objects, %d are still alive.%n", new Object[] {i, Finalizable.aliveCount.get() });
                    }
          }
     }
}
```
这个程序使用了一个无限循环来创建对象。它同时还用了一个静态变量aliveCount来跟踪一共创建了多少个实例。每创建了一个新对象，计数器会加1，一旦GC完成后调用了finalize()方法，计数器会跟着减1。

你觉得这小段代码的输出结果会是怎样的呢？由于新创建的对象很快就没人引用了，它们马上就可以被GC回收掉。因此你可能会认为这段程序可以不停的运行下去，：

>After creating 345,000,000 objects, 0 are still alive.
>After creating 345,100,000 objects, 0 are still alive.
>After creating 345,200,000 objects, 0 are still alive.
>After creating 345,300,000 objects, 0 are still alive.

显然结果并非如此。现实的结果完全不同，在我的Mac OS X的JDK 1.7.0_51上，程序大概在创建了120万个对象后就抛出java.lang.OutOfMemoryError: GC overhead limitt exceeded异常退出了。

>After creating 900,000 objects, 791,361 are still alive.
>After creating 1,000,000 objects, 875,624 are still alive.
>After creating 1,100,000 objects, 959,024 are still alive.
>After creating 1,200,000 objects, 1,040,909 are still alive.
>Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
at java.lang.ref.Finalizer.register(Finalizer.java:90)
at java.lang.Object.(Object.java:37)
at eu.plumbr.demo.Finalizable.(Finalizable.java:8)
at eu.plumbr.demo.Finalizable.main(Finalizable.java:19)

## 垃圾回收的行为

想弄清楚到底发生了什么，你得看下这段程序在运行时的状况如何。我们来打开-XX:+PrintGCDetails选项再运行一次看看：

> [GC [PSYoungGen: 16896K->2544K(19456K)] 16896K->16832K(62976K), 0.0857640 secs] [Times: user=0.22 sys=0.02, real=0.09 secs]
[GC [PSYoungGen: 19440K->2560K(19456K)] 33728K->31392K(62976K), 0.0489700 secs] [Times: user=0.14 sys=0.01, real=0.05 secs]
[GC-- [PSYoungGen: 19456K->19456K(19456K)] 48288K->62976K(62976K), 0.0601190 secs] [Times: user=0.16 sys=0.01, real=0.06 secs]
[Full GC [PSYoungGen: 16896K->14845K(19456K)] [ParOldGen: 43182K->43363K(43520K)] 60078K->58209K(62976K) [PSPermGen: 2567K->2567K(21504K)], 0.4954480 secs] [Times: user=1.76 sys=0.01, real=0.50 secs]
[Full GC [PSYoungGen: 16896K->16820K(19456K)] [ParOldGen: 43361K->43361K(43520K)] 60257K->60181K(62976K) [PSPermGen: 2567K->2567K(21504K)], 0.1379550 secs] [Times: user=0.47 sys=0.01, real=0.14 secs]
--- cut for brevity---
[Full GC [PSYoungGen: 16896K->16893K(19456K)] [ParOldGen: 43351K->43351K(43520K)] 60247K->60244K(62976K) [PSPermGen: 2567K->2567K(21504K)], 0.1231240 secs] [Times: user=0.45 sys=0.00, real=0.13 secs]
[Full GCException in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
[PSYoungGen: 16896K->16866K(19456K)] [ParOldGen: 43351K->43351K(43520K)] 60247K->60218K(62976K) [PSPermGen: 2591K->2591K(21504K)], 0.1301790 secs] [Times: user=0.44 sys=0.00, real=0.13 secs]
at eu.plumbr.demo.Finalizable.main(Finalizable.java:19)

从日志中可以看到，少数几次的Eden区的新生代GC过后，JVM开始采用更昂贵的Full GC来清理老生代和持久代的空间。为什么会这样？既然已经没有人引用这些对象了，为什么它们没有在新生代中被回收掉？代码这么写有什么问题吗？

要弄清楚GC这个行为的原因，我们先来对代码做一个小的改动，将finalize()方法的实现先去掉。现在JVM发现这个类没有实现finalize()方法了，于是它切换回了”正常”的模式。再看一眼GC的日志，你只能看到一些廉价的新生代GC在不停的运行。

![Java内存](/images/java-memory-model.png)

因为修改后的这段程序中，的确没有人引用到了新生代的这些刚创建的对象。因此Eden区很快就被清空掉了，整个程序可以一直的执行下去。

另一方面，在早先的那个例子中情况则有些不同。这些对象并非没人引用 ，JVM会为每一个Finalizable对象创建一个看门狗（watchdog）。这是Finalizer类的一个实例。而所有的这些看门狗又会为Finalizer类所引用。由于存在这么一个引用链，因此整个的这些对象都是存活的。

那现在Eden区已经满了，而所有对象又都存在引用，GC没辙了只能把它们全拷贝到Suvivor区。更糟糕的是，一旦连Survivor区也满了，只能存到老生代里面了。你应该还记得，Eden区使用的是一种”抛弃一切”的清理策略，而老生代的GC则完全不同，它采用的是一种开销更大的方式。

## Finalizer队列
只有在GC完成后，JVM才会意识到除了Finalizer对象已经没有人引用到我们创建的这些实例了，因此它才会把指向这些对象的Finalizer对象标记成可处理的。GC内部会把这些Finalizer对象放到java.lang.ref.Finalizer.ReferenceQueue这个特殊的队列里面。

完成了这些麻烦事之后，我们的应用程序才能继续往下走。这里有个线程你一定会很感兴趣——Finalizer守护线程。通过使用jstack进行thread dump可以看到这个线程的信息。

>My Precious:~ demo$ jps
1703 Jps
1702 Finalizable
My Precious:~ demo$ jstack 1702

>--- cut for brevity ---
"Finalizer" daemon prio=5 tid=0x00007fe33b029000 nid=0x3103 runnable [0x0000000111fd4000]
   java.lang.Thread.State: RUNNABLE
at java.lang.ref.Finalizer.invokeFinalizeMethod(Native Method)
at java.lang.ref.Finalizer.runFinalizer(Finalizer.java:101)
at java.lang.ref.Finalizer.access$100(Finalizer.java:32)
at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:190)
--- cut for brevity —

从上面可以看到有一个Finalizer守护线程正在运行。Finalizer线程是个单一职责的线程。这个线程会不停的循环等待java.lang.ref.Finalizer.ReferenceQueue中的新增对象。一旦Finalizer线程发现队列中出现了新的对象，它会弹出该对象，调用它的finalize()方法，将该引用从Finalizer类中移除，因此下次GC再执行的时候，这个Finalizer实例以及它引用的那个对象就可以回垃圾回收掉了。

现在我们有两个线程都在不停地循环。我们的主线程在忙着创建新对象。这些对象都有各自的看门狗也就是Finalizer，而这个Finalizer对象会被添加到一个java.lang.ref.Finalizer.ReferenceQueue中。Finalizer线程会负责处理这个队列，它将所有的对象弹出，然后调用它们的finalize()方法。

很多时候你可能磁不到内存溢出这种情况。finalize()方法的调用会比你创建新对象要早得多。因此大多数时候，Finalizer线程能够赶在下次GC带来更多的Finalizer对象前清空这个队列。但我们这个例子当中，显然不是这样。

为什么会出现溢出？因为Finalizer线程和主线程相比它的优先级要低。这意味着分配给它的CPU时间更少，因此它的处理速度没法赶上新对象创建的速度。这就是问题的根源——对象创建的速度要比Finalizer线程调用finalize()结束它们的速度要快，这导致最后堆中所有可用的空间都被耗尽了。结果就是——我们亲爱的小伙伴java.lang.OutOfMemoryError会以不同的身份出现在你面前。

如果你仍然不相信我的话，dump一下堆内存，看下它里面有什么。比如说，你可以使用-XX:+HeapDumpOnOutOfMemoryError参数启动我们这个小程序，在我的Eclipse中的MAT Dominator Tree中我看到的是下面这张图：



看到了吧，我这个64M的堆全给Finalizer对象给占满了。

结论
回顾一下，Finalizable对象的生命周期和普通对象的行为是完全不同的，列举如下：

- JVM创建Finalizable对象
- JVM创建 java.lang.ref.Finalizer实例，指向刚创建的对象。
- java.lang.ref.Finalizer类持有新创建的java.lang.ref.Finalizer的实例。这使得下一次新生代GC无法回收这些对象。
- 新生代GC无法清空Eden区，因此会将这些对象移到Survivor区或者老生代。
- 垃圾回收器发现这些对象实现了finalize()方法。因为会把它们添加到java.lang.ref.Finalizer.ReferenceQueue队列中。
- Finalizer线程会处理这个队列，将里面的对象逐个弹出，并调用它们的finalize()方法。
finalize()方法调用完后，Finalizer线程会将引用从Finalizer类中去掉，因此在下一轮GC中，这些对象就可以被回收了。
- Finalizer线程会和我们的主线程进行竞争，不过由于它的优先级较低，获取到的CPU时间较少，因此它永远也赶不上主线程的步伐。
- 程序消耗了所有的可用资源，最后抛出OutOfMemoryError异常。

这篇文章想告诉我们什么？下回如果你考虑使用finalize()方法，而不是使用常规的方式来清理对象的话，最好多想一下。你可能会为使用了finalize()方法写出的整洁的代码而沾沾自喜，但是不停增长的Finalizer队列也许会撑爆你的年老代，你需要重新再考虑一下你的方案。

参考：

1. [Debugging to Understand Finalizer
](https://dzone.com/articles/debugging-understand-finalizer)
2. [Java的Finalizer引发的内存溢出](https://www.cnblogs.com/benwu/articles/5812903.html)