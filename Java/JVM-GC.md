### 判断对象是否死亡

**引用计数算法**

单纯的引用计数 就很难解决对象之间相互循环引用的问题。

**可达性分析算法**

当前主流的商用程序语言（Java、C#，上溯至前面提到的古老的Lisp）的内存管理子系统，都是通过可达性分析（Reachability Analysis）算法来判定对象是否存活的。这个算法的基本思路就是通过 一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过 程所走过的路径称为“引用链”（Reference Chain），如果某个对象到GC Roots间没有任何引用链相连， 或者用图论的话来说就是从GC Roots到这个对象不可达时，则证明此对象是不可能再被使用的。

### 引用

在JDK 1.2版之后，Java对引用的概念进行了扩充，将引用分为强引用（Strongly Re-ference）、软 引用（Soft Reference）、弱引用（Weak Reference）和虚引用（Phantom Reference）4种，这4种引用强 度依次逐渐减弱。

- 强引用是最传统的“引用”的定义，是指在程序代码之中普遍存在的引用赋值，即类似“Object obj=new Object()”这种引用关系。无论任何情况下，只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。
- **软引用**是用来描述一些**还有用，但非必须**的对象。只被软引用关联着的对象，在系统**将要发生内存溢出异常前**，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存， 才会抛出内存溢出异常。在JDK 1.2版之后提供了SoftReference类来实现软引用。
- 弱引用也是用来描述那些非必须对象，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生为止。当垃圾收集器开始工作，**无论当前内存是否足够**，都会回收掉只 被引用关联的对象。在JDK 1.2版之后提供了WeakReference类来实现弱引用。
- 虚引用也称为“幽灵引用”或者“幻影引用”，它是最弱的一种引用关系。一个对象是否有虚引用的存在，**完全不会对其生存时间构成影响**，也无法通过虚引用来取得一个对象实例。为一个对象设置虚 引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知。在JDK 1.2版之后提供 了PhantomReference类来实现虚引用。

### 垃圾收集算法

从如何判定对象消亡的角度出发，垃圾收集算法可以划分为“引用计数式垃圾收集”（Reference Counting GC）和“追踪式垃圾收集”（Tracing GC）两大类，这两类也常被称作“直接垃圾收集”和“间接 垃圾收集”。

- 标记 清除算法
- 标记 复制算法
  把内存分为相同大小的两块，每次使用其中一块，GC时把存活的复制到另一块，原有块全部清理，适合新生代
- 标记 整理算法
  清理时让存活对象向一端移动，适合老年代，但有Stop The World问题

### 分代收集理论

建立在两个分代假说之上：

- 弱分代假说（Weak Generational Hypothesis）：绝大多数对象都是朝生夕灭的。
- 强分代假说（Strong Generational Hypothesis）：熬过越多次垃圾收集过程的对象就越难以消亡。

堆中有

- 新生代（新生代中有Eden区，Survivor区）
- 老年代
- 元空间（永久代）

新生代Eden满了之后，触发MinorGC， ~~把Eden区域和From幸存区里存活的对象**复制**到TO幸存区，然后From和TO两个幸存区互换~~

当寿命到达阈值后，把幸存区中的内容晋升到老年代

### 垃圾收集器

Serial：新生代收集器，最为基础历史最悠久的收集器；单线程，进行垃圾收集时会暂停工作线程；仍然是HotSpot虚拟机客户端模式下默认的新生代收集器；新生代采用标记复制，老年代采用标记整理

Serial Old收集器：Serial收集器的老年版本

ParNew收集器：新生代收集器，Serial收集器的多线程并行版本，在单核心处理器的效果绝对不会比Serial更好，有线程交互的开销，与CMS配合

Parallel Scavenge收集器：新生代收集器，关注**吞吐量**，可以设置最大垃圾收集停顿时间和吞吐量大小

Parallel Old收集器：Parallel Scavenge收集器的老年代版本

CMS收集器：以获取最短回收停顿时间（**响应时间**）为目标的收集器

- 初始标记 -暂停其他所有线程
- 并发标记
- 重新标记 -暂停其他所有线程，比初始标记时间长，但比并发标记时间短
- 并发清理

G1收集器
    在后台维护了一个优先列表，根据允许的收集时间优先回收价值最大的Region
    将整个JAVA堆分为2048个region块，每个region块大小由实际的堆空间大小决定，为1MB~32MB
    每个region根据需要自由选择扮演什么角色
    回收分为四个步骤：
    - 初始标记，标记GC Roots能直接关联的对象，需要停顿线程
    - 并发标记，从GC Roots开始对堆中对象进行可达性分析，但可与用户程序并发执行
    - 最终标记，对用户线程做一个短暂暂停，处理并发标记阶段漏标的对象
    - 筛选回收，负责更新Region的统计数据，根据每个Region的回收价值和成本进行排序，把决定回收的那一部分Region的存活对象复制到空Region中，再清理掉整个旧Region的整个空间，**需要暂停用户线程**

![总览](assets/Pasted%20image%2020221209102236.png)
