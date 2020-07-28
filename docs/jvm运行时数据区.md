## jvm运行时数据区

# 运行时数据区

![NX5054.png](https://s1.ax1x.com/2020/07/03/NX5054.png)

# 程序计数器

程序计数器指的是当前线程正在执行的字节码指令地址（行号），

java中最小的执行单位是线程，因为虚拟机的是多线程的，每个线程是抢夺cpu时间片，程序计数器就是存储这些指令去做什么，比如循环，跳转，异常处理等等需要依赖它

每个线程都有属于自己的程序计数器，而且互不影响，独立存储。3

# Java虚拟机栈
栈-是数据结构-存储数据

存储什么数据？存储当前线程的运行方法时所需的数据，指令，返回地址

我们知道方法运行时都会出现一个栈帧，用于存储局部变量表，操作数栈，动态链接，方法出口等

![NX5bsP.png](https://s1.ax1x.com/2020/07/03/NX5bsP.png)

![NX5xiQ.png](https://s1.ax1x.com/2020/07/03/NX5xiQ.png)

 # 本地方法栈

本地方法栈和虚拟机栈所发挥的作用是相似的。本地方法栈为虚拟机运行native方法服务，与虚拟机栈相同的是栈的深度是固定的，当线程申请的大于虚拟机栈的深度就会抛出StackOverflowError异常，当然虚拟机栈也可以动态的扩展，如果扩展到无法申请到足够的内存就会抛出out ofMemoryError异常
# 方法区

它用于存储已被虚拟机加载的类信息（class文件）、常量（1.7有变化存储在堆里面）、静态变量、即时编译器编译后的代码等数据

package com.jvm;

 

import java.util.ArrayList;

import java.util.List;

 

/**

 * -XX:MaxPermSize=10M 方法区最大大小10M

*/

public class Test {

 

    public static void main(String[] args) {

        List<String> list = new ArrayList<String>();

        int i = 0;

        while (true) {

            list.add(String.valueOf(i++).intern());   //不断创建线程

        }

    }

}

字符串常量池在JDK6的时候还是存放在方法区（永久代）所以它会抛出OutOfMemoryError:Permanent Space；而JDK7后则将字符串常量池移到了Java堆中，上面的代码不会抛出OOM，若将堆内存改为20M则会抛出OutOfMemoryError:Java heap space；至于JDK8则是纯粹取消了方法区这个概念，取而代之的是”元空间（Metaspace）“，所以在JDK8中虚拟机参数”-XX:MaxPermSize”也就没有了任何意义，取代它的是”-XX:MetaspaceSize“和”-XX:MaxMetaspaceSize”等。
# Java堆
此区域存放的是存放对象实例，java堆是所有线程共享区域的内存区域

![NXIUSA.png](https://s1.ax1x.com/2020/07/03/NXIUSA.png)

![NXIcWj.png](https://s1.ax1x.com/2020/07/03/NXIcWj.png)

堆大小=新生代+老年代；（新生代占堆空间的1/3、老年代占堆空间2/3）

新生代又被分为了eden、from survivor、to survivor(8:1:1)；

**为什么是8:1:1？**

新生代这样划分是为了更好的管理堆内存中的对象，方便GC算法---复制算法来进行垃圾回收。

JVM每次只会使用eden和其中一块survivor来为对象服务，所以无论什么时候，都会有一块survivor空间，因此新生代实际可用空间只有90%

新生代GC（minor gc）----------指发生在新生代的垃圾回收动作，因为JAVA对象大多数都是朝生夕死的特性，所以minor gc非常平凡，使用复制算法快速的回收。

新生代几乎是所有JAVA对象出生的地方，JAVA对象申请的内存和存放都是在这个地方。

当对象在eden(其中包括一个survivor，假如是from)，当此对象经过一次minor gc后仍然存活，并且能够被另外一块survivor所容纳（这里survivor则是to了），则使用复制算法将这些仍然存活的对象复制到to survior区域中，然后清理掉eden和from survivor区域，并将这些存活的对象年龄+1，以后对象在survivor中每熬过一次gc则增加1，当年龄达到某个值时（默认15，通过设置参数-xx:maxtenuringThreshold来设置），这些对象就会成为老年代！

但是也不一定，当一些较大的对象（需要分配连续的内存空间）则直接进入老年代。

老年代GC（major gc）----------指发生在老年代的垃圾回收动作，所采用是的标记--整理算法。

老年代几乎都是经过survivor熬过来的，它们是不会那么容易“死掉”，因此major gc不会想minor gc那样频繁

**JVM 新生代为何需要两个 Survivor 空间？**

我们知道，目前主流的虚拟机实现都采用了分代收集的思想，把整个堆区划分为新生代和老年代；新生代又被划分成 Eden 空间、 From Survivor 和 To Survivor 三块区域。 

看书的时候有个疑问，为什么非得是两个 Survivor 空间呢？要回答这个问题，其实等价于：为什么不是0个或1个 Survivor 空间？为什么2个 Survivor 空间可以达到要求？ 

**为什么不是0个 Survivor 空间？**

这个问题等价于：为什么需要 Survivor 空间。我们看看如果没有 Survivor 空间的话，垃圾收集将会怎样进行：一遍新生代 gc 过后，不管三七二十一，活着的对象全部进入老年代，即便它在接下来的几次 gc 过程中极有可能被回收掉。这样的话老年代很快被填满， Full GC 的频率大大增加。我们知道，老年代一般都会被规划成比新生代大很多，对它进行垃圾收集会消耗比较长的时间；如果收集的频率又很快的话，那就更糟糕了。基于这种考虑，虚拟机引进了“幸存区”的概念：如果对象在某次新生代 gc 之后任然存活，让它暂时进入幸存区；以后每熬过一次 gc ，让对象的年龄＋1，直到其年龄达到某个设定的值（比如15岁）， JVM 认为它很有可能是个“老不死的”对象，再呆在幸存区没有必要（而且老是在两个幸存区之间反复地复制也需要消耗资源），才会把它转移到老年代。 

总之，设置 Survivor 空间的目的是让那些中等寿命的对象尽量在 Minor GC 时被干掉，最终在总体上减少虚拟机的垃圾收集过程对用户程序的影响。 

**为什么不是1个 Survivor 空间？**

回答这个问题有一个前提，就是新生代一般都采用复制算法进行垃圾收集。原始的复制算法是把一块内存一分为二， gc 时把存活的对象（Eden和Survivor to）从一块空间（From space）复制到另外一块空间（To space），再把原先的那块内存（From space）清理干净，最后调换 From space 和 To space 的逻辑角色（这样下一次 gc 的时候还可以按这样的方式进行）。 

我们知道，在 HotSpot 虚拟机里， Eden 空间和 Survivor 空间默认的比例是 8:1 。我们来看看在只有一个 Survivor 空间的情况下，这个 8:1 会有什么问题。此处为了方便说明，我们假设新生代一共为 9 MB 。对象优先在 Eden 区分配，当 Eden 空间满 8 MB 时，触发第一次 Minor GC 。比如说有 0.5 MB 的对象存活，那这 0.5 MB 的对象将由 Eden 区向 Survivor 区复制。这次 Minor GC 过后， Eden 区被清理干净， Survivor 区被占用了 0.5 MB ，还剩 0.5 MB 。到这里一切都很美好，但问题马上就来了：从现在开始所有对象将会在这剩下的 0.5 MB 的空间上被分配，很快就会发现空间不足，于是只好触发下一次 Minor GC 。可以看出在这种情况下，当 Survivor 空间作为对象“出生地”的时候，很容易触发 Minor GC ，这种 8:1 的不对称分配不但没能在总体上降低 Minor GC 的频率，还会把 gc 的时间间隔搞得很不平均。把 Eden : Survivor 设成 1 : 1 也一样，每当对象总大小满 5 MB 的时候都必须触发一次 Minor GC ，唯一的变化是 gc 的时间间隔相对平均了。 

上面的论述都是以“新生代使用复制算法”这个既定事实作为前提来讨论的。如果不是这样，比如说新生代采用“标记-清除”或者“标记-整理”算法来实现幸存对象的移动，好像确实是只需要一个 Survivor 就够了。

**为什么2个 Survivor 空间可以达到要求？ **

问题很清楚了，无论 Eden 和 Survivor 的比例怎么设置，在只有一个 Survivor 的情况下，总体上看在新生代空间满一半的时候就会触发一次 Minor GC 。那有没有提升的空间呢？比如说永远在新生代空间满 80% 的时候才触发 Minor GC ？ 

事实上是可以做到的：我们可以设两个 Survivor 空间（ From Survivor 和 To Survivor ）。比如，我们把 Eden : From Survivor : To Survivor 空间大小设成 8 : 1 : 1 ，对象总是在 Eden 区出生， From Survivor 保存当前的幸存对象， To Survivor 为空。一次 gc 发生后： 
1）Eden 区活着的对象 ＋ From Survivor 存储的对象被复制到 To Survivor ； 
2) 清空 Eden 和 From Survivor ； 
3) 颠倒 From Survivor 和 To Survivor 的逻辑关系： From 变 To ， To 变 From 。 

可以看出，只有在 Eden 空间快满的时候才会触发 Minor GC 。而 Eden 空间占新生代的绝大部分，所以 Minor GC 的频率得以降低。当然，使用两个 Survivor 这种方式我们也付出了一定的代价，如 10% 的空间浪费、复制对象的开销等。

上面体现的垃圾回收算法有三种：

- 复制回收算法
- 标记清除
- 标记整理



