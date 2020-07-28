-------

synchronized的底层实现原理（偏向锁、轻量级锁、重入）
# 理解Java对象头与Monitor
在JVM中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充。如下：
![image.png](https://i.loli.net/2020/06/18/PAogUZ76xEIF2MJ.png)

- 实例变量：存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐。

- 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这点了解即可。

 而对于顶部，则是Java头对象，它实现synchronized的锁对象的基础，这点我们重点分析它，一般而言，synchronized使用的锁对象是存储在Java对象头里的，jvm中采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来的1个字记录的是数组长度)，其主要结构是由Mark Word 和 Class Metadata Address 组成，其结构说明如下表

![image.png](https://i.loli.net/2020/06/18/5Uu1vGNWk7P8BLK.png)

其中Mark Word在默认情况下存储着对象的HashCode、分代年龄、锁标记位等以下是32位JVM的Mark Word默认存储结构

![image.png](https://i.loli.net/2020/06/18/zJm61ZMDEKCpOyR.png)

由于对象头的信息是与对象自身定义的数据没有关系的额外存储成本，因此考虑到JVM的空间效率，Mark Word 被设计成为一个非固定的数据结构，以便存储更多有效的数据，它会根据对象本身的状态复用自己的存储空间，如32位JVM下，除了上述列出的Mark Word默认存储结构外，还有如下可能变化的结构：

![image.png](https://i.loli.net/2020/06/18/y6buwMzO9iXBAE3.png)

  monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因(关于这点稍后还会进行分析)，ok~，有了上述知识基础后，下面我们将进一步分析synchronized在字节码层面的具体语义实现。

# synchronized同步块底层原理      
同步语句块的实现使用的是monitorenter 和 monitorexit 指令，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置，当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。synchronized同步块对同一条线程来说式可重入的，不会出现自己把自己锁死的情况。如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor (关于重入性稍后会分析)，重入时计数器的值也会加 1。倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。

值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都有执行其对应 monitorexit 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时 monitorenter 和 monitorexit 指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。从字节码中也可以看出多了一个monitorexit指令，它就是异常结束时被执行的释放monitor 的指令。

# synchronized方法底层原理
   方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放。
 

    public class SyncMethod {
     
       public int i;
     
       public synchronized void syncTask(){
       i++;
       }
    }
使用javap反编译后的字节码如下：

    Classfile /Users/zejian/Downloads/Java8_Action/src/main/java/com/zejian/concurrencys/SyncMethod.class
      Last modified 2017-6-2; size 308 bytes
      MD5 checksum f34075a8c059ea65e4cc2fa610e0cd94
      Compiled from "SyncMethod.java"
    public class com.zejian.concurrencys.SyncMethod
      minor version: 0
      major version: 52
      flags: ACC_PUBLIC, ACC_SUPER
    Constant pool;
     
       //省略没必要的字节码
      //==================syncTask方法======================
      public synchronized void syncTask();
    descriptor: ()V
    //方法标识ACC_PUBLIC代表public修饰，ACC_SYNCHRONIZED指明该方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
     0: aload_0
     1: dup
     2: getfield  #2  // Field i:I
     5: iconst_1
     6: iadd
     7: putfield  #2  // Field i:I
    10: return
      LineNumberTable:
    line 12: 0
    line 13: 10
    }
    SourceFile: "SyncMethod.java"
   从字节码中可以看出，synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取得代之的确实是ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法，JVM通过该ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。这便是synchronized锁在同步代码块和同步方法上实现的基本原理。

# 背景--线程状态切换的代价
   java的线程是映射到操作系统原生线程之上的，如果要阻塞或唤醒一个线程就需要操作系统介入，需要在户态与核心态之间切换，这种切换会消耗大量的系统资源，因为用户态与内核态都有各自专用的内存空间，专用的寄存器等，用户态切换至内核态需要传递给许多变量、参数给内核，内核也需要保护好用户态在切换时的一些寄存器值、变量等，以便内核态调用结束后切换回用户态继续工作。

   因为需要限制不同的程序之间的访问能力, 防止他们获取别的程序的内存数据, 或者获取外围设备的数据, 并发送到网络, CPU划分出两个权限等级 :用户态 和 内核态



- 内核态：CPU可以访问内存所有数据, 包括外围设备, 例如硬盘, 网卡. CPU也可以将自己从一个程序切换到另一个程序


- 用户态：只能受限的访问内存, 且不允许访问外围设备. 占用CPU的能力被剥夺, CPU资源可以被其他程序获取

   所有用户程序都是运行在用户态的, 但是有时候程序确实需要做一些内核态的事情, 例如从硬盘读取数据, 或者从键盘获取输入等.，而唯一可以做这些事情的就是操作系统, 所以此时程序就需要先操作系统请求以程序的名义来执行这些操作（比如java的I/O操作底层都是通过native方法来调用操作系统）。这时需要一个这样的机制: 用户态程序切换到内核态, 但是不能控制在内核态中执行的指令。这种机制叫系统调用, 在CPU中的实现称之为陷阱指令(Trap Instruction)。参考详情

   synchronized会导致争用不到锁的线程进入阻塞状态，所以说它是java语言中一个重量级的同步操纵，被称为重量级锁，为了缓解上述性能问题，JVM从1.5开始，引入了轻量锁与偏向锁，默认启用了自旋锁，他们都属于乐观锁。所以明确java线程切换的代价，是理解java中各种锁的优缺点的基础之一。

   同时我们还必须注意到的是在Java早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，挂起线程和恢复线程都需要转入内核态去完成，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。庆幸的是在Java 6之后Java官方对从JVM层面对synchronized较大优化，所以现在的synchronized锁效率也优化得很不错了，Java 6之后，为了减少获得锁和释放锁所带来的性能消耗，引入了轻量级锁和偏向锁，接下来我们将简单了解一下Java官方在JVM层面对synchronized锁的优化。

# Java虚拟机对synchronized的优化
锁的状态总共有四种，无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁，但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级，关于重量级锁，前面我们已详细分析过，下面我们将介绍偏向锁和轻量级锁以及JVM的其他优化手段。

##偏向锁
步骤如下：



- 一个线程去争用时，如果没有其他线程争用，则会尝试CAS去修改mark word中一个标记为偏向(mark word单独有一个bit表示是否可偏向，记录锁的位置依然为01)，这个CAS动作同时会修改mark word部分bit以保留线程的ID值。


- 当线程不断发生重入时，只需要判定头部的线程ID是否是当前线程，若是，则无需任何操作。


- 如果同一个对象存在另一个线程发起了访问请求，则首先会判定该对象是否已经被锁定了。如果已经被锁定，则会将锁修改为轻量级锁(00),也就是锁粒度上升了；而如果没有锁定，则会将对象的是否可偏向的位置设置为不可偏向。

偏向锁是Java 6之后加入的新锁，它是一种针对加锁操作的优化手段，经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了减少同一线程获取锁(会涉及到一些CAS操作,耗时)的代价而引入偏向锁。偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提供程序的性能。所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。下面我们接着了解轻量级锁。

CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。也可以保证原子性

 如果当前内存位置的值等于预期原值A的话，就将B赋值。否则，处理器不做任何操作。整个比较并替换的操作是一个原子操作。这样做就不用害怕其它线程同时修改变量。

## 轻量级锁
 synchronized会在对象的头部打标记，这个加锁的动作是必须要做的，悲观锁通常还会做许多其他的指令动作，轻量级锁希望通过CAS实现，它认为通过CAS尝试修改对象头部的mark区域的内容就可以达到目的，由于mark区域的宽度通常是4~8字节，也就是相当于一个int或者long的宽度，是否适合于CAS操作。

轻量级锁通常会做一下4个步骤：



- 在栈中分配一块空间用来做一份对象头部mark word的拷贝，在mark word中将对象锁的二进制位设置为“未锁定”(在32位的JVM中通常有2位用于存储锁标记，未锁定的标记为01)，这个动作是方便等到释放锁的时候将这份数据拷贝到对象头部。


- 通过CAS尝试将头部的二进制位修改为“线程私有栈中对mark区域拷贝存放的地址”，如果成功，则会将最后2位设置为00，代表已经被轻量级锁锁住了。


- 如果没有成功，则判定对象头部是否已经指向了当前线程所在的栈当中，如果成立则代表当前线程已经是拥有着，可以继续执行。


- 如果不是拥有着，则说明有多个线程在争用，那么此时会将锁升级为悲观锁，线程进入BLOCKED状态。

JVM发现在轻量级锁里面多次“重入”和“释放”时，需要做的判断和拷贝动作还是很多，而在某些应用程序中，锁就是被某一个线程一直使用，为了进一步减小锁的开销，JVM中出现了偏向锁，偏向锁希望记录的是一个线程ID，它比轻量级锁更加轻量，当再次重入判定时，首先判定对象头部的线程ID是不是当前线程，若是则表示当前线程已经是对象锁的OWNER，无须做其他任何动作。

## 自旋锁
 
轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。这是基于在大多数情况下，线程持有锁的时间都不会太长，如果直接挂起操作系统层面的线程可能会得不偿失，毕竟操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，因此自旋锁会假设在不久将来，当前的线程可以获得锁，因此虚拟机会让当前想要获取锁的线程做几个空循环(这也是称为自旋的原因)。问题在于，自旋是需要消耗CPU的，如果一直获取不到锁的话，那该线程就一直处在自旋状态，白白浪费CPU资源。解决这个问题最简单的办法就是给线程空循环设置一个次数，当线程超过了这个次数，我们就认为，继续使用自旋锁就不适合了，此时锁会再次膨胀，升级为重量级锁。

## 自适应自旋锁
所谓自适应自旋锁就是线程空循环等待的自旋次数并非是固定的，而是会动态着根据实际情况来改变自旋等待的次数。简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。

## 锁粗化
锁粗化的概念应该比较好理解，就是将多次连接在一起的加锁、解锁操作合并为一次，将多个连续的锁扩展成一个范围更大的锁。举个例子：

    package com.paddx.test.string;
     
    public class StringBufferTest {
    StringBuffer stringBuffer = new StringBuffer();
     
    public void append(){
    stringBuffer.append("a");
    stringBuffer.append("b");
    stringBuffer.append("c");
    }
    }
这里每次调用stringBuffer.append方法都需要加锁和解锁，如果虚拟机检测到有一系列连串的对同一个对象加锁和解锁操作，就会将其合并成一次范围更大的加锁和解锁操作，即在第一次append方法时进行加锁，最后一次append方法结束后进行解锁。

假如有一个循环，循环内的操作需要加锁，我们应该把锁放到循环外面，否则每次进出循环，都进出一次临界区，效率是非常差的；

## 锁消除
锁消除即删除不必要的加锁操作。根据代码逃逸技术，如果判断到一段代码中，堆上的数据不会逃逸出当前线程，那么可以认为这段代码是线程安全的，不必要加锁。看下面这段程序：

	package com.paddx.test.concurrent;
 
		public class SynchronizedTest02 {
 
	    public static void main(String[] args) {
	        SynchronizedTest02 test02 = new SynchronizedTest02();
	        //启动预热
	        for (int i = 0; i < 10000; i++) {
	            i++;
	        }
	        long start = System.currentTimeMillis();
	        for (int i = 0; i < 100000000; i++) {
	            test02.append("abc", "def");
	        }
	        System.out.println("Time=" + (System.currentTimeMillis() - start));
	    }
 
	    public void append(String str1, String str2) {
	        StringBuffer sb = new StringBuffer();
	        sb.append(str1).append(str2);
	    }
	}



虽然StringBuffer的append是一个同步方法，但是这段程序中的StringBuffer属于一个局部变量，并且不会从该方法中逃逸出去，所以其实这过程是线程安全的，可以将锁消除。

偏向锁是在无锁争用的情况下使用的，也就是同步开在当前线程没有执行完之前，没有其它线程会执行该同步快，一旦有了第二个线程的争用，偏向锁就会升级为轻量级锁，一点有两个以上线程争用，就会升级为重量级锁；如果线程争用激烈，那么应该禁用偏向锁。

## 锁重入
即在使用synchronized时，当一个线程得到一个对象锁后，再次请求此对象锁时，是可以再次得到该对象的锁的。也就是说在一个synchronized方法或块的内部调用本类的其他synchronized方法或块时，是永远可以得到锁的。 可重入锁即自己可以再次获取自己的内部锁，反之不可重入锁就造成死锁了

	class Service{
	    public synchronized void service1(){
	        System.out.println("service 1 ");
	        service2();
	    }
	 
	    public synchronized void service2(){
	        System.out.println("service 2");
	        service3();
	    }
	 
	    public synchronized void service3(){
	        System.out.println("service 3");
	    }
	}
	 
	class MyThread extends Thread{
	    @Override
	    public void run() {
	        Service service = new Service();
	        service.service1();
	    }
	}
	 
	public class Run {
	    public static void main(String[] args) {
	        MyThread t = new MyThread();
	        t.start();
	    }
	}
# 线程中断与synchronized
## 线程中断

正如中断二字所表达的意义，在线程运行(run方法)中间打断它，在Java中，提供了以下3个有关线程中断的方法

    //中断线程（实例方法）
    public void Thread.interrupt();
    
    //判断线程是否被中断（实例方法）
    public boolean Thread.isInterrupted();
    
    //判断是否被中断并清除当前中断状态（静态方法）
    public static boolean Thread.interrupted();

当一个线程处于被阻塞状态或者试图执行一个阻塞操作时，使用Thread.interrupt()方式中断该线程，注意此时将会抛出一个InterruptedException的异常，同时中断状态将会被复位(由中断状态改为非中断状态)，如下代码将演示该过程：

	public class InterruputSleepThread3 {
	    public static void main(String[] args) throws InterruptedException {
	        Thread t1 = new Thread() {
	            @Override
	            public void run() {
	                //while在try中，通过异常中断就可以退出run循环
	                try {
	                    while (true) {
	                        //当前线程处于阻塞状态，异常必须捕捉处理，无法往外抛出
	                        TimeUnit.SECONDS.sleep(2);
	                    }
	                } catch (InterruptedException e) {
	                    System.out.println("Interruted When Sleep");
	                    boolean interrupt = this.isInterrupted();
	                    //中断状态被复位
	                    System.out.println("interrupt:"+interrupt);
	                }
	            }
	        };
	        t1.start();
	        TimeUnit.SECONDS.sleep(2);
	        //中断处于阻塞状态的线程
	        t1.interrupt();
	
	        /**
	         * 输出结果:
	           Interruted When Sleep
	           interrupt:false
	         */
	    }
	}

如上述代码所示，我们创建一个线程，并在线程中调用了sleep方法从而使用线程进入阻塞状态，启动线程后，调用线程实例对象的interrupt方法中断阻塞异常，并抛出InterruptedException异常，此时中断状态也将被复位。这里有些人可能会诧异，为什么不用Thread.sleep(2000);而是用TimeUnit.SECONDS.sleep(2);其实原因很简单，前者使用时并没有明确的单位说明，而后者非常明确表达秒的单位，事实上后者的内部实现最终还是调用了Thread.sleep(2000);，但为了编写的代码语义更清晰，建议使用TimeUnit.SECONDS.sleep(2);的方式，注意TimeUnit是个枚举类型。ok~，除了阻塞中断的情景，我们还可能会遇到处于运行期且非阻塞的状态的线程，这种情况下，直接调用Thread.interrupt()中断线程是不会得到任响应的，如下代码，将无法中断非阻塞状态下的线程：

	public class InterruputThread {
	    public static void main(String[] args) throws InterruptedException {
	        Thread t1=new Thread(){
	            @Override
	            public void run(){
	                while(true){
	                    System.out.println("未被中断");
	                }
	            }
	        };
	        t1.start();
	        TimeUnit.SECONDS.sleep(2);
	        t1.interrupt();
	
	        /**
	         * 输出结果(无限执行):
	             未被中断
	             未被中断
	             未被中断
	             ......
	         */
	    }
	}
虽然我们调用了interrupt方法，但线程t1并未被中断，因为处于非阻塞状态的线程需要我们手动进行中断检测并结束程序，改进后代码如下：

	public class InterruputThread {
	    public static void main(String[] args) throws InterruptedException {
	        Thread t1=new Thread(){
	            @Override
	            public void run(){
	                while(true){
	                    //判断当前线程是否被中断
	                    if (this.isInterrupted()){
	                        System.out.println("线程中断");
	                        break;
	                    }
	                }
	
	                System.out.println("已跳出循环,线程中断!");
	            }
	        };
	        t1.start();
	        TimeUnit.SECONDS.sleep(2);
	        t1.interrupt();
	
	        /**
	         * 输出结果:
	            线程中断
	            已跳出循环,线程中断!
	         */
	    }
	}
是的，我们在代码中使用了实例方法isInterrupted判断线程是否已被中断，如果被中断将跳出循环以此结束线程,注意非阻塞状态调用interrupt()并不会导致中断状态重置。综合所述，可以简单总结一下中断两种情况，一种是当线程处于阻塞状态或者试图执行一个阻塞操作时，我们可以使用实例方法interrupt()进行线程中断，执行中断操作后将会抛出interruptException异常(该异常必须捕捉无法向外抛出)并将中断状态复位，另外一种是当线程处于运行状态时，我们也可调用实例方法interrupt()进行线程中断，但同时必须手动判断中断状态，并编写中断线程的代码(其实就是结束run方法体的代码)。有时我们在编码时可能需要兼顾以上两种情况，那么就可以如下编写：

	public void run(){
	    try {
	    //判断当前线程是否已中断,注意interrupted方法是静态的,执行后会对中断状态进行复位
	    while (!Thread.interrupted()) {
	        TimeUnit.SECONDS.sleep(2);
	    }
	    } catch (InterruptedException e) {
	
	    }
	}
## 中断与synchronized
事实上线程的中断操作对于正在等待获取的锁对象的synchronized方法或者代码块并不起作用，也就是对于synchronized来说，如果一个线程在等待锁，那么结果只有两种，要么它获得这把锁继续执行，要么它就保存等待，即使调用中断线程的方法，也不会生效。演示代码如下

	/**
	 * Created by zejian on 2017/6/2.
	 * Blog : http://blog.csdn.net/javazejian [原文地址,请尊重原创]
	 */
	public class SynchronizedBlocked implements Runnable{
	
	    public synchronized void f() {
	        System.out.println("Trying to call f()");
	        while(true) // Never releases lock
	            Thread.yield();
	    }
	
	    /**
	     * 在构造器中创建新线程并启动获取对象锁
	     */
	    public SynchronizedBlocked() {
	        //该线程已持有当前实例锁
	        new Thread() {
	            public void run() {
	                f(); // Lock acquired by this thread
	            }
	        }.start();
	    }
	    public void run() {
	        //中断判断
	        while (true) {
	            if (Thread.interrupted()) {
	                System.out.println("中断线程!!");
	                break;
	            } else {
	                f();
	            }
	        }
	    }
	
	
	    public static void main(String[] args) throws InterruptedException {
	        SynchronizedBlocked sync = new SynchronizedBlocked();
	        Thread t = new Thread(sync);
	        //启动后调用f()方法,无法获取当前实例锁处于等待状态
	        t.start();
	        TimeUnit.SECONDS.sleep(1);
	        //中断线程,无法生效
	        t.interrupt();
	    }
	}

我们在SynchronizedBlocked构造函数中创建一个新线程并启动获取调用f()获取到当前实例锁，由于SynchronizedBlocked自身也是线程，启动后在其run方法中也调用了f()，但由于对象锁被其他线程占用，导致t线程只能等到锁，此时我们调用了t.interrupt();但并不能中断线程。

等待唤醒机制与synchronized
所谓等待唤醒机制本篇主要指的是notify/notifyAll和wait方法，在使用这3个方法时，必须处于synchronized代码块或者synchronized方法中，否则就会抛出IllegalMonitorStateException异常，这是因为调用这几个方法前必须拿到当前对象的监视器monitor对象，也就是说notify/notifyAll和wait方法依赖于monitor对象，在前面的分析中，我们知道monitor 存在于对象头的Mark Word 中(存储monitor引用指针)，而synchronized关键字可以获取 monitor ，这也就是为什么notify/notifyAll和wait方法必须在synchronized代码块或者synchronized方法调用的原因。

	synchronized (obj) {
	       obj.wait();
	       obj.notify();
	       obj.notifyAll();         
	 }
