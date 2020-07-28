----

# Java线程池原理解读

引言




> 引用自《阿里巴巴JAVA开发手册》
> 【强制】线程资源必须通过线程池提供，不允许在应用中自行显式创建线程。

> 说明：使用线程池的好处是减少在创建和销毁线程上所消耗的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者“过度切换”的问题。

之前在阅读《阿里巴巴JAVA开发手册》时发现，其中有一条对于线程资源的值用限制，要求使用线程池来创建和维护，那么什么是线程池呢，为什么是线程池？原理是什么？怎么使用它？有什么讲究呢？带着这一系列的问题，我们开始来探究一下，希望这篇文章对我们有所收获。

本文源码来自JDK 1.8 。

# 简介
线程池，故名思意，就是一个存放线程的池子，学术一点的说法，就是一组存放线程资源的集合。为什么有线程池这一概念地产生呢？想想以前我们都是需要线程的时候，直接自己手动来创建一个，然后执行完任务我们就不管了，线程就是我们执行异步任务的一个工具或者说载体，我们并没有太多关注于这个线程自身生命周期对于系统或环境的影响，而只把重心放在了多线程任务执行完成的结果输出，然后目的达到了，但是真正忽略了线程资源的维护和监控等问题。随着大型系统大量多线程资源的使用，对多线程疏于重视、维护和管理而对资源占用和拉低性能的影响逐渐扩大，才引起了人们的思考。多线程的创建和销毁在多线程的生命周期中占有很大比重，这一部分其实很占用资源和性能，如果使用线程来执行简单任务，而因为线程本身的维护成本已经超出任务执行的效益，这是得不偿失的，于是就产生了线程池。通过使用线程池，将线程的生命周期管控起来，同时能够方便地获取到线程、复用线程，避免频繁地创建和销毁线程带来额外性能开销，这大概就是线程池引入的背景和初衷吧。

所以现在看来，合理的利用线程池能够给系统带来几大好处：

1、减低资源消耗。通过重复利用已创建好的线程来降低线程创建和销毁造成的消耗；

2、提高响应速度。当任务到达时，任务可以不需要等待线程创建就能立马执行；

3、提高线程可管理性。线程池时稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

但是，如果想合理的掌控线程池的使用，那么多线程池的原理和特性，都是必须要了解清楚的。

# 解读
API继承结构图
JDK为我们提供一个叫做Excutor的框架来使用线程池，它是线程池的基础，我们可以看下线程池相关的Java Diagram：

![imageffaaeb14f99fbc84.png](https://wx1.sbimg.cn/2020/06/19/imageffaaeb14f99fbc84.png)

我们从上面的图一个一个来看，分析下各个接口和类的功能和所承担的角色。

**Executor接口：**

它是线程池的基础，提供了唯一的一个方法execute用来执行线程任务。

**ExecutorService接口：**

它继承了Executor接口，提供了线程池生命周期管理的几乎所有方法，包括诸如shutdown、awaitTermination、submit、invokeAll、invokeAny等。

**AbstractExecutorService类：**

一个提供了线程池生命周期默认实现的抽象类，并且自行新增了如newTaskFor、doInvokeAny等方法。

**ThreadPoolExecutor类：**

这是线程池的核心类，也是我们常用来创建和管理线程池的类，我们使用Executors调用newFixedThreadPool、newSingleThreadExecutor和newCachedThreadPool等方法创建的线程池，都是ThreadPoolExecutor类型的。

**ScheduledExecutorService接口：**

赋予了线程池具备延迟和定期执行任务的能力，它提供了一些方法接口，使得任务能够按照给定的方式来延期或者周期性的执行任务。

**ScheduledThreadPoolExecutor类：**

继承自ThreadPoolExecutor类，同时实现了ScheduledExecutorService接口，具备了线程池所有通用能力，同时增加了延时执行和周期性执行任务的能力。

除了上面说到的这些，JDK1.7中还新增了一个线程池ForkJoinPool，它与ThreadPoolExecutor一样继承于AbstractExecutorService。与其他类型的ExecutorService相比，它的不同之处在于采用了工作窃取算法（work-stealing，可以从源码和注释中得到更多详细介绍）：所有线程池中的线程会尝试找到并执行已被提交到池中的或由其他线程创建的任务。这样的算法使得很少有线程处于空闲状态，非常的高效，这样的方式常用于如大多数由任务产生大量子任务的情况，以及像从外部客户端大量提交小任务到池中的情况。

以上是对于线程池API继承体系的简单梳理和介绍，接下来我们深入源码去进行分析。

线程池的几种内部状态
线程池使用了一个Integer类型变量来记录线程池任务数量和线程池状态信息，很巧妙。

	 private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	 private static final int COUNT_BITS = Integer.SIZE - 3;
	 private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
	 
	 // runState is stored in the high-order bits
	 private static final int RUNNING    = -1 << COUNT_BITS;
	 private static final int SHUTDOWN   =  0 << COUNT_BITS;
	 private static final int STOP       =  1 << COUNT_BITS;
	 private static final int TIDYING    =  2 << COUNT_BITS;
	 private static final int TERMINATED =  3 << COUNT_BITS;
	 
	 // Packing and unpacking ctl
	 private static int runStateOf(int c)     { return c & ~CAPACITY; }
	 private static int workerCountOf(int c)  { return c & CAPACITY; }
	 private static int ctlOf(int rs, int wc) { return rs | wc; }
看这个变量ctl，被定义为了AtomicInteger，使用高3位来表示线程池状态，低29位来表示线程池中的任务数量。

## 线程池状态
**RUNNING**：线程池能够接受新任务，以及对新添加的任务进行处理。

**SHUTDOWN**：线程池不可以接受新任务，但是可以对已添加的任务进行处理。

**STOP**：线程池不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。

**TIDYING**：当所有的任务已终止，ctl记录的"任务数量"为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现。

**TERMINATED**：线程池彻底终止的状态。

根据代码设计，我们用图标来展示一下

![imageabb218181ea91092.png](https://wx1.sbimg.cn/2020/06/19/imageabb218181ea91092.png)

各线程池状态的切换图示

![image4e3056d1783991ec.png](https://wx2.sbimg.cn/2020/06/19/image4e3056d1783991ec.png)

# 原理分析
原理分析，我们将结合源码的和注释的方式来分析。

**核心参数**
通过上面的描述我们知道，线程池的核心实现即ThreadPoolExecutor类就是本次我们重点关注和学习的对象。从对它的初始化过程我们看到，它完整的构造方法向我们暴露了这几个核心的参数：

	public ThreadPoolExecutor(int corePoolSize,
	                          int maximumPoolSize,
	                          long keepAliveTime,
	                          TimeUnit unit,
	                          BlockingQueue<Runnable> workQueue,
	                          ThreadFactory threadFactory,
	                          RejectedExecutionHandler handler) {
	    if (corePoolSize < 0 ||
	        maximumPoolSize <= 0 ||
	        maximumPoolSize < corePoolSize ||
	        keepAliveTime < 0)
	        throw new IllegalArgumentException();
	    if (workQueue == null || threadFactory == null || handler == null)
	        throw new NullPointerException();
	    this.corePoolSize = corePoolSize;
	    this.maximumPoolSize = maximumPoolSize;
	    this.workQueue = workQueue;
	    this.keepAliveTime = unit.toNanos(keepAliveTime);
	    this.threadFactory = threadFactory;
	    this.handler = handler;
	}

通过上面构造函数可以看出，ThreadPoolExecutor类具有7个参数，由于篇幅受限我没有把构造函数的注释文档贴上来，我现在逐个翻译并简要说明一下：

**corePoolSize**：核心线程数，当线程数小于该值时，线程池会优先创建新线程来执行任务，如果调用了线程池的prestartAllCoreThreads方法，线程池会提前创建并启动所有基本线程，除非设置了allowCoreThreadTimeOut，否则核心线程将持续保留在线程池中即时没有新的任务提交过来。

**maximumPoolSize**：最大线程数，即线程池所能允许创建的最大线程数量。

**keepAliveTime**：空闲线程存活时间，当线程数量大于核心线程数时，这是多余空闲线程在终止之前等待新任务的最长时间。

**unit**：keepAliveTime数值的时间单位。

**workQueue**：任务队列，用于缓存未执行的任务，队列一直会持有任务直到有线程开始执行它。

**threadFactory**：线程工厂，可以通过工厂创建更具识别性质的线程，如线程名字等。

**handler**：拒绝策略，当线程和队列都处于饱和时就使用拒绝策略来处理新任务。

# 线程池中线程的使用和创建规则
在Java线程池的实现逻辑中，线程池所能创建的线程数量受限于corePoolSize和maximumPoolSize两个参数值，线程的创建时机则和corePoolSize和workQueue两个参数相关，当线程数量和队列长度都已达到饱和时，则介入拒绝策略来处理新的任务了，下面把大概的流程说明一下。

1、当来了新任务，如果线程池中空闲线程数量小于corePoolSize，则直接拿线程池中新的线程来处理任务；

2、如果线程池正在运行的线程数量大于等于corePoolSize，而此时workQueue队列未满，则将此任务缓存到队列中；

3、如果线程池正在运行的线程数量大于等于corePoolSize，且workQueue队列已满，但现在的线程数量还小于maximumPoolSize，则创建新的线程来执行任务。

4、如果线程数量已经大于maximumPoolSize且workQueue队列也已经满了，则使用拒绝策略来处理该任务，默认的拒绝策略就是抛出异常（AbortPolicy）。

我们简化下上面的文字，用简单表格来展示：



我们再用流程图来对线程池中提交任务的这一逻辑增加感性认识：

![image6f6178937bae4d96.png](https://wx1.sbimg.cn/2020/06/19/image6f6178937bae4d96.png)

下面，我们通过代码，来重点性的解读一下这一流程：

## 1、线程的创建和复用
线程池中线程的创建是通过线程工厂ThreadFactory来实现的，线程池的默认实现是使用Executors.defaultThreadFactory()来返回的工厂类，我们可以通过构造函数指定线程工厂，这里不做深入了解。

顺便提一句，线程池的创建其实还有个关键方法prestartAllCoreThreads()，它的作用就是在线程池刚初始化的时候就激活核心线程数大小的线程放置到线程池中，等待任务着任务来执行，但是JDK默认的启动策略中并没有使用它，我按照这个方法查询了一下，在Tomcat包中实现的ThreadPoolExecutor中在构造的时候，都调用了这个方法来初始化核心线程数量。

线程复用是线程池作用的关键所在，避免线程重复创建和销毁，重复使用空闲的未销毁的线程。所以这就要求一个线程在执行完一个任务之后不能直接退出，需要重新去队列任务中获取新的任务来执行，如果任务队列中没有任务，且keepAliveTime没有被设置，那么这个工作线程将一直阻塞下去指导有新的任务可执行，这样就达到了线程复用的目的。

	private final class Worker
	    extends AbstractQueuedSynchronizer
	    implements Runnable
	{
	    // 创建任务调用内部类Worker
	    Worker(Runnable firstTask) {
	        setState(-1); // inhibit interrupts until runWorker
	        this.firstTask = firstTask;
	        // 通过线程工厂来创建线程
	        this.thread = getThreadFactory().newThread(this);
	    }
	
	    // 实现了Runnable接口
	    public void run() {
	        runWorker(this);
	    }
	}
	
	// 执行Worker任务
	final void runWorker(Worker w) {
	    Thread wt = Thread.currentThread();
	    Runnable task = w.firstTask;
	    w.firstTask = null;
	    w.unlock(); // allow interrupts
	    boolean completedAbruptly = true;
	    try {
	        // 循环从队列中获取任务
	        while (task != null || (task = getTask()) != null) {
	            w.lock();
	            // If pool is stopping, ensure thread is interrupted;
	            // if not, ensure thread is not interrupted.  This
	            // requires a recheck in second case to deal with
	            // shutdownNow race while clearing interrupt
	            if ((runStateAtLeast(ctl.get(), STOP) ||
	                 (Thread.interrupted() &&
	                  runStateAtLeast(ctl.get(), STOP))) &&
	                !wt.isInterrupted())
	                wt.interrupt();
	            try {
	                beforeExecute(wt, task);
	                Throwable thrown = null;
	                try {
	                    // 执行线程任务
	                    task.run();
	                } catch (RuntimeException x) {
	                    thrown = x; throw x;
	                } catch (Error x) {
	                    thrown = x; throw x;
	                } catch (Throwable x) {
	                    thrown = x; throw new Error(x);
	                } finally {
	                    afterExecute(task, thrown);
	                }
	            } finally {
	                task = null;
	                w.completedTasks++;
	                w.unlock();
	            }
	        }
	        completedAbruptly = false;
	    } finally {
	        processWorkerExit(w, completedAbruptly);
	    }
	}

## 2、提交线程任务
提交任务调用线程池的submit方法，该方法在AbstractExecutorService中。
	
	public Future<?> submit(Runnable task) {
	    if (task == null) throw new NullPointerException();
	    // 创建任务
	    RunnableFuture<Void> ftask = newTaskFor(task, null);
	    // 执行任务
	    execute(ftask);
	    return ftask;
	}

其中的execute接口在Executor接口中定义，具体的实现在ThreadPoolExecutor中得以体现。

	public void execute(Runnable command) {
	    if (command == null)
	        throw new NullPointerException();
	    /*
	     * Proceed in 3 steps:
	     *
	     * 1. If fewer than corePoolSize threads are running, try to
	     * start a new thread with the given command as its first
	     * task.  The call to addWorker atomically checks runState and
	     * workerCount, and so prevents false alarms that would add
	     * threads when it shouldn't, by returning false.
	     *
	     * 2. If a task can be successfully queued, then we still need
	     * to double-check whether we should have added a thread
	     * (because existing ones died since last checking) or that
	     * the pool shut down since entry into this method. So we
	     * recheck state and if necessary roll back the enqueuing if
	     * stopped, or start a new thread if there are none.
	     *
	     * 3. If we cannot queue task, then we try to add a new
	     * thread.  If it fails, we know we are shut down or saturated
	     * and so reject the task.
	     */
	     // ctl记录了线程数量和线程状态
	    int c = ctl.get();
	    // 如果工作线程数小于核心线程数，创建新线程执行
	    // 即时其他线程是空闲的
	    if (workerCountOf(c) < corePoolSize) {
	        if (addWorker(command, true))
	            return;
	        c = ctl.get();
	    }
	    // 缓存任务到队列中，这里进行了double check
	    // 如果线程池中运行的线程数量>=corePoolSize，
	    // 且线程池处于RUNNING状态，且把提交的任务成功放入阻塞队列中，
	    // 就再次检查线程池的状态。
	    // 1.如果线程池不是RUNNING状态，且成功从阻塞队列中删除任务，
	    //   则该任务由当前 RejectedExecutionHandler 处理。
	    // 2.否则如果线程池中运行的线程数量为0，则通过
	    //   addWorker(null, false)尝试新建一个线程，
	    //   新建线程对应的任务为null。
	    if (isRunning(c) && workQueue.offer(command)) {
	        int recheck = ctl.get();
	        // 任务无效则拒绝
	        if (! isRunning(recheck) && remove(command))
	            reject(command);
	        else if (workerCountOf(recheck) == 0)
	            addWorker(null, false);
	    }
	    // 添加新的工作线程，并在addWorker方法中的两个for循环来判断
	    // 如果以上两个条件不成立，既没能将任务成功放入阻塞队列中，
	    // 且addWoker新建线程失败，则该任务由当前
	    // RejectedExecutionHandler 处理。
	    else if (!addWorker(command, false))
	        // 采用拒绝策略
	        reject(command);
	}

addWorker方法用于新增任务，第二个boolean参数表示线程数是否控制在核心线程数之内。

	private boolean addWorker(Runnable firstTask, boolean core) {
	    retry:
	    for (;;) {
	        int c = ctl.get();
	        int rs = runStateOf(c);
	
	        // Check if queue empty only if necessary.
	        if (rs >= SHUTDOWN &&
	            ! (rs == SHUTDOWN &&
	               firstTask == null &&
	               ! workQueue.isEmpty()))
	            return false;
	
	        for (;;) {
	            int wc = workerCountOf(c);
	            if (wc >= CAPACITY ||
	                wc >= (core ? corePoolSize : maximumPoolSize))
	                return false;
	            if (compareAndIncrementWorkerCount(c))
	                break retry;
	            c = ctl.get();  // Re-read ctl
	            if (runStateOf(c) != rs)
	                continue retry;
	            // else CAS failed due to workerCount change; retry inner loop
	        }
	    }
	
	    boolean workerStarted = false;
	    boolean workerAdded = false;
	    Worker w = null;
	    try {
	        // 创建工作线程
	        w = new Worker(firstTask);
	        final Thread t = w.thread;
	        if (t != null) {
	            final ReentrantLock mainLock = this.mainLock;
	            mainLock.lock();
	            try {
	                // Recheck while holding lock.
	                // Back out on ThreadFactory failure or if
	                // shut down before lock acquired.
	                int rs = runStateOf(ctl.get());
	
	                if (rs < SHUTDOWN ||
	                    (rs == SHUTDOWN && firstTask == null)) {
	                    if (t.isAlive()) // precheck that t is startable
	                        throw new IllegalThreadStateException();
	                    // 将线程放置于HashSet中，持有mainLock才可访问
	                    // workers中记录了池中真正任务线程数量
	                    workers.add(w);
	                    int s = workers.size();
	                    if (s > largestPoolSize)
	                        largestPoolSize = s;
	                    workerAdded = true;
	                }
	            } finally {
	                mainLock.unlock();
	            }
	            if (workerAdded) {
	                t.start();
	                workerStarted = true;
	            }
	        }
	    } finally {
	        if (! workerStarted)
	            addWorkerFailed(w);
	    }
	    return workerStarted;
	}

## 3、关闭线程池
ThreadPoolExecutor提供了shutdown()和shutdownNow()两个方法来关闭线程。

shutdown：

将线程状态设置为SHUTDOWN，同时中断线程，按过去执行已提交任务的顺序，发起一个有序的关闭命令，而且不再接收新的任务，最后尝试将线程池状态设置为TERMINATED。

shutdownNow：

将线程状态设置为STOP，中断所有的任务且不再接收新任务，尝试停止所有正在执行的任务、暂停等待处理的任务，并返回等待执行的任务列表。中断线程使用Thread.interrupt方法，未响应中断命令的任务是无法被中断的。

# JDK提供的常用的线程池
一般情况下我们都不直接用ThreadPoolExecutor类来创建线程池，而是通过Executors工具类去构建，通过Executors工具类我们可以构造5种不同的线程池。

**newFixedThreadPool(int nThreads)：**

创建固定线程数的线程池，corePoolSize和maximumPoolSize是相等的，默认情况下，线程池中的空闲线程不会被回收的；

**newCachedThreadPool：**

创建线程数量不定的线程池，线程数量随任务量变动，一旦来了新的任务，如果线程池中没有空闲线程则立马创建新的线程来执行任务，空闲线程存活时间60秒，过后就被回收了，可见这个线程池弹性很高；

**newSingleThreadExecutor：**

创建线程数量为1的线程池，等价于newFixedThreadPool(1)所构造的线程池；

**newScheduledThreadPool(int corePoolSize)：**

创建核心线程数为corePoolSize，可执行定时任务的线程池；

**newSingleThreadScheduledExecutor：**

等价于newScheduledThreadPool(1)。

## 阻塞队列
构造函数中的队列允许我们自定义，队列的意义在于缓存无法得到线程执行的任务，当线程数量大于corePoolSize而当前workQueue还没有满时，就需要将任务放置到队列中。JDK提供了几种类型的队列容器，每种类型都具各自特点，可以根据实际场景和需要自行配置到线程池中。

**ArrayBlockingQueue：**

有界队列，基于数组结构，按照队列FIFO原则对元素排序；

**LinkedBlockingQueue：**

无界队列，基于链表结构，按照队列FIFO原则对元素排序，Executors.newFixedThreadPool()使用了这个队列；

**SynchronousQueue：**

同步队列，该队列不存储元素，每个插入操作必须等待另一个线程调用移除操作，否则插入操作会一直被阻塞，Executors.newCachedThreadPool()使用了这个队列；

**PriorityBlockingQueue：**

优先级队列，具有优先级的无限阻塞队列。

## 拒绝策略
拒绝策略（RejectedExecutionHandler）也称饱和策略，当线程数量和队列都达到饱和时，就采用饱和策略来处理新提交过来的任务，默认情况下采用的策略是抛出异常（AbortPolicy），表示无法处理直接抛出异常，其实JDK提供了四种策略，也很好记，拒绝策略无非就是抛异常、执行或者丢弃任务，其中丢弃任务就分为丢弃自己或者丢弃队列中最老的任务，下面简要说明一下：

**AbortPolicy**：丢弃新任务，并抛出 RejectedExecutionException

**DiscardPolicy**：不做任何操作，直接丢弃新任务

**DiscardOldestPolicy**：丢弃队列队首（最老）的元素，并执行新任务

**CallerRunsPolicy**：由当前调用线程来执行新任务

## 使用技巧
使用了线程池技术未必能够给工作带来利好，在没能正确理解线程池特性以及了解自身业务场景下而配置的线程池，可能会成为系统性能或者业务的瓶颈甚至是漏洞，所以在我们使用线程池时，除了对线程池本身特性了如指掌，还需要对自身业务属性进行一番分析，以便配置出合理的高效的线程池以供项目使用，下面我们从这几个方面来分析：

- 任务的性质：CPU密集型任务，IO密集型任务和混合型任务。
- 任务的优先级：高，中和低。
- 任务的执行时间：长，中和短。
- 任务的依赖性：是否依赖其他系统资源，如数据库连接。

性质不同的任务可以用不同规模的线程池分开处理。CPU密集型任务配置尽可能少的线程数量，如配置Ncpu+1个线程的线程池，以减少线程切换带来的性能开销。IO密集型任务则由于需要等待IO操作，线程并不是一直在执行任务，则配置尽可能多的线程，如2*Ncpu。混合型的任务，如果可以拆分，则将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐率要高于串行执行的吞吐率，如果这两个任务执行时间相差太大，则没必要进行分解。我们可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。

优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理。它可以让优先级高的任务先得到执行，需要注意的是如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行。

当然，以上这些配置方式都是经验值，实际当中还需要分析自己的项目场景经过多次测试方可得出最适合自己项目的线程池配置。

## 自问自答
面试的时候，可能会遇到面试官问下面的这样几个问题，看看你现在能不能回答，算是一种学习的自我检查吧。

1、线程池原理，参数如何设置？

2、线程池有哪些参数，阻塞队列用的是什么队列，为什么？

3、线程池原理，为什么要创建线程池，创建线程池的方式？

4、创建线程池有哪几个核心参数，如何合理配置线程池大小？