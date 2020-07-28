# AQS概述
​	AbstractQueuedSynchronizer抽象队列同步器简称AQS，它是实现同步器的基础组件，juc下面Lock的实现以及一些并发工具类就是通过AQS来实现的，这里我们通过AQS的类图先看一下大概，下面我们总结一下AQS的实现原理。先看看AQS的类图。
![image.png](https://i.loli.net/2020/06/18/4IJLm95QU73NXbT.png)


​(1)AQS是一个通过内置的FIFO双向队列来完成线程的排队工作(内部通过结点head和tail记录队首和队尾元素，元素的结点类型为Node类型，后面我们会看到Node的具体构造)。

    /*等待队列的队首结点(懒加载，这里体现为竞争失败的情况下，加入同步队列的线程执行到enq方法的时候会创
    建一个Head结点)。该结点只能被setHead方法修改。并且结点的waitStatus不能为CANCELLED*/
    private transient volatile Node head;
    /**等待队列的尾节点，也是懒加载的。（enq方法）。只在加入新的阻塞结点的情况下修改*/
    private transient volatile Node tail;
​	(2)其中Node中的thread用来存放进入AQS队列中的线程引用，Node结点内部的SHARED表示标记线程是因为获取共享资源失败被阻塞添加到队列中的；Node中的EXCLUSIVE表示线程因为获取独占资源失败被阻塞添加到队列中的。waitStatus表示当前线程的等待状态：

​	①CANCELLED=1：表示线程因为中断或者等待超时，需要从等待队列中取消等待；

​	②SIGNAL=-1：当前线程thread1占有锁，队列中的head(仅仅代表头结点，里面没有存放线程引用)的后继结点node1处于等待状态，如果已占有锁的线程thread1释放锁或被CANCEL之后就会通知这个结点node1去获取锁执行。

​	③CONDITION=-2：表示结点在等待队列中(这里指的是等待在某个lock的condition上，关于Condition的原理下面会写到)，当持有锁的线程调用了Condition的signal()方法之后，结点会从该condition的等待队列转移到该lock的同步队列上，去竞争lock。(注意：这里的同步队列就是我们说的AQS维护的FIFO队列，等待队列则是每个condition关联的队列)

​	④PROPAGTE=-3：表示下一次共享状态获取将会传递给后继结点获取这个共享同步状态。

(3)AQS中维持了一个单一的volatile修饰的状态信息state(AQS通过Unsafe的相关方法，以原子性的方式由线程去获取这个state)。AQS提供了getState()、setState()、compareAndSetState()函数修改值(实际上调用的是unsafe的compareAndSwapInt方法)。下面是AQS中的部分成员变量以及更新state的方法

    //这就是我们刚刚说到的head结点，懒加载的（只有竞争失败需要构建同步队列的时候，才会创建这个head），如果头节点存在，它的waitStatus不能为CANCELLED
    private transient volatile Node head;
    //当前同步队列尾节点的引用，也是懒加载的，只有调用enq方法的时候会添加一个新的wait node
    private transient volatile Node tail;
    //AQS核心：同步状态
    private volatile int state;
    protected final int getState() {
    return state;
    }
    protected final void setState(int newState) {
    state = newState;
    }
    protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
​	(4)AQS的设计师基于模板方法模式的。使用时候需要继承同步器并重写指定的方法，并且通常将子类推荐为定义同步组件的静态内部类，子类重写这些方法之后，AQS工作时使用的是提供的模板方法，在这些模板方法中调用子类重写的方法。其中子类可以重写的方法：

    //独占式的获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行CAS设置同步状态
    protected boolean tryAcquire(int arg) {	throw new UnsupportedOperationException();}
    //独占式的释放同步状态，等待获取同步状态的线程可以有机会获取同步状态
    protected boolean tryRelease(int arg) {	throw new UnsupportedOperationException();}
    //共享式的获取同步状态
    protected int tryAcquireShared(int arg) { throw new UnsupportedOperationException();}
    //尝试将状态设置为以共享模式释放同步状态。 该方法总是由执行释放的线程调用。 
    protected int tryReleaseShared(int arg) { throw new UnsupportedOperationException(); }
    //当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占
    protected int isHeldExclusively(int arg) {	throw new UnsupportedOperationException();}
(5)AQS的内部类ConditionObject是通过结合锁实现线程同步，ConditionObject可以直接访问AQS的变量(state、queue)，ConditionObject是个条件变量 ，每个ConditionObject对应一个队列用来存放线程调用condition条件变量的await方法之后被阻塞的线程。


# AQS中的独占模式
​	上面我们简单了解了一下AQS的基本组成，这里通过ReentrantLock的非公平锁实现来具体分析AQS的独占模式的加锁和释放锁的过程。


# 非公平锁的加锁流程
​	简单说来，AQS会把所有的请求线程构成一个CLH队列，当一个线程执行完毕（lock.unlock()）时会激活自己的后继节点，但正在执行的线程并不在队列中，而那些等待执行的线程全部处于阻塞状态(park())。如下图所示。
![image.png](https://i.loli.net/2020/06/18/DZNLJX9cKGVl6s5.png)


​(1)假设这个时候在初始情况下，还没有多任务来请求竞争这个state，这时候如果第一个线程thread1调用了lock方法请求获得锁，首先会通过CAS的方式将state更新为1，表示自己thread1获得了锁，并将独占锁的线程持有者设置为thread1。

    final void lock() {
    if (compareAndSetState(0, 1))
    //setExclusiveOwnerThread是AbstractOwnableSynchronizer的方法，AQS继承了AbstractOwnableSynchronizer
    setExclusiveOwnerThread(Thread.currentThread());
    else
    acquire(1);
    }
​(2)这个时候有另一个线程thread2来尝试或者锁，同样也调用lock方法，尝试通过CAS的方式将state更新为1，但是由于之前已经有线程持有了state，所以thread2这一步CAS失败（前面的thread1已经获取state并且没有释放），就会调用acquire(1)方法（该方法是AQS提供的模板方法，它会调用子类的tryAcquire方法）。非公平锁的实现中，AQS的模板方法acquire(1)就会调用NofairSync的tryAcquire方法，而tryAcquire方法又调用的Sync的nonfairTryAcquire方法，所以我们看看nonfairTryAcquire的流程。
    
    //NofairSync
    protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
    }
    final boolean nonfairTryAcquire(int acquires) {
    //（1）获取当前线程
    final Thread current = Thread.currentThread();
    //（2）获得当前同步状态state
    int c = getState();
    //（3）如果state==0，表示没有线程获取
    if (c == 0) {
    //（3-1）那么就尝试以CAS的方式更新state的值
    if (compareAndSetState(0, acquires)) {
    //（3-2）如果更新成功，就设置当前独占模式下同步状态的持有者为当前线程
    setExclusiveOwnerThread(current);
    //（3-3）获得成功之后，返回true
    return true;
    }
    }
    //（4）这里是重入锁的逻辑
    else if (current == getExclusiveOwnerThread()) {
    //（4-1）判断当前占有state的线程就是当前来再次获取state的线程之后，就计算重入后的state
    int nextc = c + acquires;
    //（4-2）这里是风险处理
    if (nextc < 0) // overflow
    throw new Error("Maximum lock count exceeded");
    //（4-3）通过setState无条件的设置state的值，（因为这里也只有一个线程操作state的值，即
    //已经获取到的线程，所以没有进行CAS操作）
    setState(nextc);
    return true;
    }
    //（5）没有获得state，也不是重入，就返回false
    return false;
    }
总结来说就是：

1、获取当前将要去获取锁的线程thread2。

2、获取当前AQS的state的值。如果此时state的值是0，那么我们就通过CAS操作获取锁，然后设置AQS的线程占有者为thread2。很明显，在当前的这个执行情况下，state的值是1不是0，因为我们的thread1还没有释放锁。所以CAS失败，后面第3步的重入逻辑也不会进行

3、如果当前将要去获取锁的线程等于此时AQS的exclusiveOwnerThread的线程，则此时将state的值加1，这是重入锁的实现方式。

4、最终thread2执行到这里会返回false。

​(3)上面的thread2加锁失败，返回false。那么根据开始我们讲到的AQS概述就应该将thread2构造为一个Node结点加入同步队列中。因为NofairSync的tryAcquire方法是由AQS的模板方法acquire()来调用的，那么我们看看该方法的源码以及执行流程。

    //(1)tryAcquire，这里thread2执行返回了false，那么就会执行addWaiter将当前线程构造为一个结点加入同步队列中
    public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
    }
​	那么我们就看一下addWaiter方法的执行流程。

    private Node addWaiter(Node mode) {
    //(1)将当前线程以及阻塞原因(是因为SHARED模式获取state失败还是EXCLUSIVE获取失败)构造为Node结点
    Node node = new Node(Thread.currentThread(), mode);
    //(2)这一步是快速将当前线程插入队列尾部
    Node pred = tail;
    if (pred != null) {
    //(2-1)将构造后的node结点的前驱结点设置为tail
    node.prev = pred;
    //(2-2)以CAS的方式设置当前的node结点为tail结点
    if (compareAndSetTail(pred, node)) {
    //(2-3)CAS设置成功，就将原来的tail的next结点设置为当前的node结点。这样这个双向队
    //列就更新完成了
    pred.next = node;
    return node;
    }
    }
    //(3)执行到这里，说明要么当前队列为null，要么存在多个线程竞争失败都去将自己设置为tail结点，
    //那么就会有线程在上面（2-2）的CAS设置中失败，就会到这里调用enq方法
    enq(node);
    return node;
    }
​	那么总结一下add Waiter方法

​	1、将当前将要去获取锁的线程也就是thread2和独占模式封装为一个node对象。

​	2、尝试快速的将当前线程构造的node结点添加作为tail结点(这里就是直接获取当前tail，然后将node的前驱结点设置为tail)，并且以CAS的方式将node设置为tail结点(CAS成功后将原tail的next设置为node，然后这个队列更新成功)。

​	3、如果2设置失败，就进入enq方法。

​	在刚刚的thread1和thread2的环境下，开始时候线程阻塞队列是空的(因为thread1获取了锁，thread2也是刚刚来请求锁，所以线程阻塞队列里面是空的)。很明显，这个时候队列的尾部tail节点也是null，那么将直接进入到enq方法。所以我们看看enq方法的实现

    private Node enq(final Node node) {
    for (;;) {
    //(4)还是先获取当前队列的tail结点
    Node t = tail;
    //(5)如果tail为null，表示当前同步队列为null，就必须初始化这个同步队列的head和tail（建
    //立一个哨兵结点）
    if (t == null) { 
    //（5-1）初始情况下，多个线程竞争失败，在检查的时候都发现没有哨兵结点，所以需要CAS的
    //设置哨兵结点
    if (compareAndSetHead(new Node()))
    tail = head;
    } 
    //(6)tail不为null
    else {
    //(6-1)直接将当前结点的前驱结点设置为tail结点
    node.prev = t;
    //(6-2)前驱结点设置完毕之后，还需要以CAS的方式将自己设置为tail结点，如果设置失败，
    //就会重新进入循环判断一遍
    if (compareAndSetTail(t, node)) {
    t.next = node;
    return t;
    }
    }
    }
    }
​	enq方法内部是一个自旋循环，第一次循环默认情况如下图所示

​	1、首先代码块（4）处将t指向了tail，判断得到t==null，如图(1)所示；

​	2、于是需要新建一个哨兵结点作为整个同步队列的头节点(代码块5-1处执行)

​	3、完了之后如图(2)所示。这样第一次循环执行完毕。
![image.png](https://i.loli.net/2020/06/18/oSUwkY8lLhDz4XB.png)


​	第二次循环整体执行如下图所示。

​	1、还是先获取当前tail结点然后将t指向tail结点。如下图的(3)

​	2、然后判断得到当前t!=null，所以enq方法中进入代码块(6).

​	3、在(6-1)代码块中将node的前驱结点设置为原来队列的tail结点，如下图的(4)所示。

​	4、设置完前驱结点之后，代码块(6-2)会以CAS的方式将当前的node结点设置为tail结点,如果设置成功，就会是下图(5)所示。更新完tail结点之后，需要保证双向队列的，所以将原来的指向哨兵结点的t的next结点指向node结点，如下图(6)所示。最后返回。

![image.png](https://i.loli.net/2020/06/18/JW5w74FNQrMBgqx.png)

​	总结来说，即使在多线程情况下，enq方法还是能够保证每个线程结点会被安全的添加到同步队列中，因为enq通过CAS方式将结点添加到同步队列之后才会返回，否则就会不断尝试添加(这样实际上就是在并发情况下，把向同步队列添加Node变得串行化了)

​	(4)在上面AQS的模板方法中，acquire()方法还有一步acquireQueued，这个方法的主要作用就是在同步队列中嗅探到自己的前驱结点，如果前驱结点是头节点的话就会尝试取获取同步状态，否则会先设置自己的waitStatus为-1，然后调用LockSupport的方法park自己。具体的实现如下面代码所示

    final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
    boolean interrupted = false;
    //在这样一个循环中尝试tryAcquire同步状态
    for (;;) {
    //获取前驱结点
    final Node p = node.predecessor();
    //(1)如果前驱结点是头节点，就尝试取获取同步状态，这里的tryAcquire方法相当于还是调
    //用NofairSync的tryAcquire方法，在上面已经说过
    if (p == head && tryAcquire(arg)) {
    //如果前驱结点是头节点并且tryAcquire返回true，那么就重新设置头节点为node
    setHead(node);
    p.next = null; //将原来的头节点的next设置为null，交由GC去回收它
    failed = false;
    return interrupted;
    }
    //(2)如果不是头节点,或者虽然前驱结点是头节点但是尝试获取同步状态失败就会将node结点
    //的waitStatus设置为-1(SIGNAL),并且park自己，等待前驱结点的唤醒。至于唤醒的细节
    //下面会说到
    if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    interrupted = true;
    }
    } finally {
    if (failed)
    cancelAcquire(node);
    }
    }
​	在上面的代码中我们可以看出，这个方法也是一个自旋循环，继续按照刚刚的thread1和thread2这个情况分析。在enq方法执行完之后，同步队列的情况大概如下所示。
![image.png](https://i.loli.net/2020/06/18/sU9LPM2aNdhG6I8.png)


​	当前的node结点的前驱结点为head，所以会调用tryAcquire()方法去获得同步状态。但是由于state被thread1占有，所以tryAcquire失败。这里就是执行acquireQueued方法的代码块(2)了。代码块(2)中首先调用了shouldParkAfterFailedAcquire方法，该方法会将同步队列中node结点的前驱结点的waitStatus为CANCELLED的线程移除，并将当前调用该方法的线程所属结点自己和他的前驱结点的waitStatus设置为-1(SIGNAL)，然后返回。具体方法实现如下所示。

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //（1）获取前驱结点的waitStatus
    int ws = pred.waitStatus;
    //（2）如果前驱结点的waitStatus为SINGNAL，就直接返回true
    if (ws == Node.SIGNAL)
    //前驱结点的状态为SIGNAL，那么该结点就能够安全的调用park方法阻塞自己了。
    return true;
    if (ws > 0) {
    //（3）这里就是将所有的前驱结点状态为CANCELLED的都移除
    do {
    node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
    } else {
    //CAS操作将这个前驱节点设置成SIGHNAL。
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
    }
​	所以shouldParkAfterFailedAcquire方法执行完毕，现在的同步队列情况大概就是这样子，即哨兵结点的waitStatus值变为-1。
![image.png](https://i.loli.net/2020/06/18/oWjpRzN24wJeUMf.png)


​	上面的执行完毕返回到acquireQueued方法的时候，在acquireQueued方法中就会进行第二次循环了，但是还是获取state失败，而当再次进入shouldParkAfterFailedAcquire方法的时候，当前结点node的前驱结点head的waitStatus已经为-1(SIGNAL)了，就会返回true，然后acquireQueued方法中就会接着执行parkAndCheckInterrupt将自己park阻塞挂起。

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
​	(5)我们梳理一下整个方法调用的流程，假设现在又有一个thread3线程竞争这个state，那么这个方法调用的流程是什么样的呢。

​	①首先肯定是调用ReentrantLock.lock()方法去尝试加锁;

​	②因为是非公平锁，所以就会转到调用NoFairSync.lock()方法;

​	③在NoFairSync.lock()方法中，会首先尝试设置state的值，因为已经被占有那么肯定就是失败的。这时候就会调用AQS的模板方法AQS.acquire(1)。

​	④在AQS的模板方法acquire(1)中，实际首先会调用的是子类的tryAcquire()方法，而在非公平锁的实现中即Sync.nofairTryAcquire()方法。

​	⑤显然tryAcquire()会返回false，所以acquire()继续执行，即调用AQS.addWaiter()，就会将当前线程构造称为一个Node结点,初始状况下waitStatus为0。

​	⑥在addWaiter方法中，会首先尝试直接将构建的node结点以CAS的方式(存在多个线程尝试将自己设置为tail)设置为tail结点，如果设置成功就直接返回，失败的话就会进入一个自旋循环的过程。即调用enq()方法。最终保证自己成功被添加到同步队列中。

​	⑦加入同步队列之后，就需要将自己挂起或者嗅探自己的前驱结点是否为头结点以便尝试获取同步状态。即调用acquireQueued()方法。

​	⑧在这里thread3的前驱结点不是head结点，所以就直接调用shouldParkAfterFailedAcquire()方法，该方法首先会将刚刚的thread2线程结点中的waitStatue的值改变为-1(初始的时候是没有改变这个waitStatus的，每个新节点的添加就会改变前驱结点的waitStatus值)。

​	⑨thread2所在结点的waitStatus改变后，shouldParkAfterFailedAcquire方法会返回false。所以之后还会在acquireQueued中进行第二次循环。并再次调用shouldParkAfterFailedAcquire方法，然后返回true。最终调用parkAndCheckInterrupt()将自己挂起。

​	每个线程去竞争这个同步状态失败的话大概就会经历上面的这些过程。假设现在thread3经历上面这些过程之后也进入同步队列，那么整个同步队列大概就是下面这样了.
![image.png](https://i.loli.net/2020/06/18/GoLriPtDydbmZg4.png)


​	将上面的流程整理一下大概就是下面这个图

![image.png](https://i.loli.net/2020/06/18/cs2GViRTv9tbrln.png)

# 非公平锁的释放流程
​	上面说一ReentrantLock为例到了怎样去获得非公平锁，那么thread1获取锁，执行完释放锁的流程是怎样的呢。首先肯定是在finally中调用ReentrantLock.unlock()方法，所以我们就从这个方法开始看起。

​	(1)从下面的unlock方法中我们可以看出，实际上是调用AQS的release()方法，其中传递的参数为1，表示每一次调用unlock方法都是释放所获得的一次state。重入的情况下会多次调用unlock方法，也保证了lock和unlock是成对的。

    public void unlock() {
    sync.release(1); //这里ReentrantLock的unlock方法调用了AQS的release方法
    }
    public final boolean release(int arg) {
    	//这里调用了子类的tryRelease方法，即ReentrantLock的内部类Sync的tryRelease方法
    if (tryRelease(arg)) {
    Node h = head;
    if (h != null && h.waitStatus != 0)
    unparkSuccessor(h);
    return true;
    }
    return false;
    }
​	(2)上面看到release方法首先会调用ReentrantLock的内部类Sync的tryRelease方法。而通过下面代码的分析，大概知道tryRelease做了这些事情。

​	①获取当前AQS的state，并减去1；

​	②判断当前线程是否等于AQS的exclusiveOwnerThread，如果不是，就抛IllegalMonitorStateException异常，这就保证了加锁和释放锁必须是同一个线程；

​	③如果(state-1)的结果不为0，说明锁被重入了，需要多次unlock，这也是lock和unlock成对的原因；

​	④如果(state-1)等于0，我们就将AQS的ExclusiveOwnerThread设置为null；

​	⑤如果上述操作成功了，也就是tryRelase方法返回了true；返回false表示需要多次unlock。

    protected final boolean tryRelease(int releases) {
    //（1）获取当前的state，然后减1，得到要更新的state
    int c = getState() - releases;
    //（2）判断当前调用的线程是不是持有锁的线程，如果不是抛出IllegalMonitorStateException
    if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
    boolean free = false;
    //（3）判断更新后的state是不是0
    if (c == 0) {
    free = true;
    //（3-1）将当前锁持者设为null
    setExclusiveOwnerThread(null);
    }
    //（4）设置当前state=c=getState()-releases
    setState(c);
    //（5）只有state==0，才会返回true
    return free;
    }
​	(3)那么当tryRelease返回true之后，就会执行release方法中if语句块中的内容。从上面我们看到，

if (tryRelease(arg)) {
    //（1）获取当前队列的头节点head
    Node h = head;
    //（2）判断头节点不为null，并且头结点的waitStatus不为0(CACCELLED)
    if (h != null && h.waitStatus != 0)
        //（3-1）调用下面的方法唤醒同步队列head结点的后继结点中的线程
        unparkSuccessor(h);
    return true;
}
​	(4)在获取锁的流程分析中，我们知道当前同步队列如下所示，所以判断得到head!=null并且head的waitStatus=-1。所以会执行unparkSuccessor方法，传递的参数为指向head的一个引用h.那下面我们就看看unparkSuccessor方法中处理了什么事情。
![image.png](https://i.loli.net/2020/06/18/mB5XNEU12OdTFtu.png)


    private void unparkSuccessor(Node node) {
    //（1）获得node的waitStatus
    int ws = node.waitStatus;
    //（2）判断waitStatus是否小于0
    if (ws < 0)
    //（2-1）如果waitStatus小于0需要将其以CAS的方式设置为0
    compareAndSetWaitStatus(node, ws, 0);
    
    //（2）获得s的后继结点，这里即head的后继结点
    Node s = node.next;
    //（3）判断后继结点是否已经被移除，或者其waitStatus==CANCELLED
    if (s == null || s.waitStatus > 0) {
    //（3-1）如果s！=null，但是其waitStatus=CANCELLED需要将其设置为null
    s = null;
    //（3-2）会从尾部结点开始寻找，找到离head最近的不为null并且node.waitStatus的结点
    for (Node t = tail; t != null && t != node; t = t.prev)
    if (t.waitStatus <= 0)
    s = t;
    }
    //（4）node.next!=null或者找到的一个离head最近的结点不为null
    if (s != null)
    //（4-1）唤醒这个结点中的线程
    LockSupport.unpark(s.thread);
    }
​	从上面的代码实现中可以总结，unparkSuccessor主要做了两件事情:

​	①获取head节点的waitStatus，如果小于0，就通过CAS操作将head节点的waitStatus修改为0

​	②寻找head节点的下一个节点，如果这个节点的waitStatus小于0，就唤醒这个节点，否则遍历下去，找到第一个waitStatus<=0的节点，并唤醒。

​	(5)下面我们应该分析的是释放掉state之后，唤醒同步队列中的结点之后程序又是是怎样执行的。按照上面的同步队列示意图，那么下面会执行这些

​	①thread1(获取到锁的线程)调用unlock方法之后，最终执行到unparkSuccessor方法会唤醒thread2结点。所以thread2被unpark。

​	②再回想一下，当时thread2是在调用acquireQueued方法之后的parkAndCheckInterrupt里面被park阻塞挂起了，所以thread2被唤醒之后继续执行acquireQueued方法中的for循环（到这里可以往前回忆看一下acquireQueued方法中的for循环做了哪些事情）；

​	③for循环中做的第一件事情就是查看自己的前驱结点是不是头结点（按照上面的同步队列情况是满足的）；

​	④前驱结点是head结点，就会调用tryAcquire方法尝试获取state，因为thread1已经释放了state，即state=0，所以thread2调用tryAcquire方法时候，以CAS的方式去将state从0更新为1是成功的，所以这个时候thread2就获取到了锁

​	⑤thread2获取state成功，就会从acquireQueued方法中退出。注意这时候的acquireQueued返回值为false，所以在AQS的模板方法的acquire中会直接从if条件退出，最后执行自己锁住的代码块中的程序。